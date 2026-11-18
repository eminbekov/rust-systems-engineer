<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.11 Project: a DNS resolver](11-project-dns-resolver.md) · [📋 Phase 15](README.md) · [15.13 QUIC and HTTP/3 →](13-quic-and-http3.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/12-tls-in-rust-servers.md) <!-- nav -->

# Lesson 15.12 — TLS in Rust servers with rustls

> **Phase 15 — Networking** · Lesson 12 of 16 · ⏱ about 135 minutes

## What you will learn

- What a TLS server needs: a certificate chain, a private key and a configuration, and what the client needs to trust it.
- How to create a small certificate authority and certificates for local tests with `rcgen`.
- How to accept TLS connections in a Tokio server with `tokio-rustls`, and how to see SNI and ALPN on the server side.
- How to write a Rust client that trusts exactly your own CA, and what its errors mean.
- How mutual TLS works: the server checks a client certificate too.

## Before you start

- You finished [Lesson 15.11 — Project: a DNS resolver](11-project-dns-resolver.md).
- You know from [Lesson 10.11 — TLS and certificates](../10-security-fundamentals/11-tls-and-certificates.md) what TLS (Transport Layer Security) protects, what the handshake does, how a chain of trust works and how a `rustls` *client* checks a server. This lesson does not repeat that; it builds the **server** side. Keep the Homebrew OpenSSL from that lesson first in your `PATH`.
- You know the Tokio accept loop from [Lesson 15.6 — Async networking with Tokio](06-async-networking-with-tokio.md).
- Create the practice package. It needs certificate files that must never go into Git:

```bash
cd ~/projects/rust
cargo new code/p15-tls
cd code/p15-tls
mkdir -p src/bin && rm src/main.rs
echo "certs/" >> .gitignore
cargo add tokio@1 --features full
cargo add rustls@0.23 --no-default-features --features ring,std,tls12
cargo add tokio-rustls@0.26 --no-default-features --features ring,tls12
cargo add rcgen@0.14 time@0.3
```

The `--no-default-features --features ring,…` lines choose one cryptography back end, `ring`, for both crates, as in Lesson 10.11. "Common mistakes" shows what happens when two back ends end up in one program.

## New words

| Word | Meaning |
|---|---|
| **TLS** (Transport Layer Security) | The protocol that encrypts and authenticates a connection (Lesson 10.11). |
| **TCP / HTTP / HTTPS** (Transmission Control Protocol / HyperText Transfer Protocol / HTTP Secure) | The reliable byte stream under TLS / the web protocol of Lesson 15.9 / HTTP carried inside TLS. |
| **CA** (Certificate Authority) | Whoever signs certificates. Here you are your own small CA, trusted only by your own test clients. |
| **certificate chain** | The server's own (leaf) certificate followed by any intermediate certificates, which the server sends during the handshake. |
| **private key** | The secret half of the server's key pair. It proves that the server owns the certificate. It never leaves the server. |
| **PEM** (Privacy-Enhanced Mail) | The text format for certificates and keys, between `-----BEGIN …-----` lines (Lesson 10.11). |
| **SAN** (Subject Alternative Name) | The list of names and IP (Internet Protocol) addresses a certificate is valid for. |
| **SNI** (Server Name Indication) | A field in the client's first handshake message that says which host name it wants. One server can then pick the right certificate for many names. |
| **ALPN** (Application-Layer Protocol Negotiation) | A field in the handshake in which the client offers protocols (`h2`, `http/1.1`) and the server picks one. |
| **acceptor / connector** | The `tokio-rustls` objects that turn a `TcpStream` into a TLS stream on the server side / on the client side. |
| **mutual TLS** (mTLS) | TLS in which the client also presents a certificate and the server verifies it. Common between internal services. |
| **crypto provider** | The library that does the actual cryptography for `rustls`: `ring` or `aws-lc-rs`. |
| **CN / DER / PKI** (Common Name / Distinguished Encoding Rules / public key infrastructure) | The main name field of a certificate / the binary encoding of certificates / everything around certificates: CAs, keys and trust. |
| **ECDSA** (Elliptic Curve Digital Signature Algorithm) | The signature algorithm of the keys that `rcgen` creates by default. |
| **close_notify** | The TLS message "I am done sending". A peer that closes without it may have been cut off by an attacker. |
| **URL** (Uniform Resource Locator) | A web address such as `https://localhost:8443/`: scheme, host, port and path. |

## 1. What a TLS server needs

In Lesson 10.11 the client did the checking: is the certificate signed by a root I trust, is it still valid, is my host name in its SAN? For those checks to pass, the **server** must provide three things:

```text
            server side                                        client side
 ┌───────────────────────────────┐                 ┌─────────────────────────────┐
 │ certificate chain (server.pem)│ ── sent in ───► │ trusted roots (ca.pem)      │
 │ private key       (server.key)│    handshake    │ expected name ("localhost") │
 │ config: versions, ALPN, mTLS  │                 │ config: versions, ALPN      │
 └───────────────────────────────┘                 └─────────────────────────────┘
        the key never leaves              the client checks: signature chain → root,
        the server                         dates, name in SAN
```

On the public internet, a public CA (for example Let's Encrypt) signs your certificate, and every client already trusts that CA. For local development and tests you do not want to touch any system trust store. Instead you create **your own CA**, sign a server certificate with it, and give the CA certificate only to your own test clients.

> 🔒 **Security:** the private key is the identity of your server. Whoever has it can pretend to be your server. Keep key files out of Git (`certs/` is in `.gitignore`), readable only by the service user, and out of container images; in production they come from a secret store (Lesson 10.8).

## 2. A local CA and certificates with rcgen

`rcgen` creates certificates in Rust code. Create `src/bin/certgen.rs`:

```rust
use std::fs;

use rcgen::{
    BasicConstraints, CertificateParams, CertifiedIssuer, DnType, IsCa, KeyPair, KeyUsagePurpose,
};
use time::{Duration, OffsetDateTime};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::create_dir_all("certs")?;
    let now = OffsetDateTime::now_utc();

    // 1. Our own certificate authority (CA). It signs the other two certificates.
    let mut ca_params = CertificateParams::new(Vec::<String>::new())?;
    ca_params
        .distinguished_name
        .push(DnType::CommonName, "p15 test CA");
    ca_params.is_ca = IsCa::Ca(BasicConstraints::Unconstrained);
    ca_params.key_usages = vec![KeyUsagePurpose::KeyCertSign];
    ca_params.not_before = now;
    ca_params.not_after = now + Duration::days(30);
    let ca = CertifiedIssuer::self_signed(ca_params, KeyPair::generate()?)?;

    // 2. The server certificate: valid for the name "localhost" and the IP 127.0.0.1.
    let mut server =
        CertificateParams::new(vec!["localhost".to_string(), "127.0.0.1".to_string()])?;
    server
        .distinguished_name
        .push(DnType::CommonName, "localhost");
    server.not_before = now;
    server.not_after = now + Duration::days(7);
    let server_key = KeyPair::generate()?;
    let server_cert = server.signed_by(&server_key, &ca)?;

    // 3. A client certificate, used for mutual TLS in Section 5.
    let mut client = CertificateParams::new(Vec::<String>::new())?;
    client
        .distinguished_name
        .push(DnType::CommonName, "client-1");
    client.not_before = now;
    client.not_after = now + Duration::days(7);
    let client_key = KeyPair::generate()?;
    let client_cert = client.signed_by(&client_key, &ca)?;

    fs::write("certs/ca.pem", ca.pem())?;
    fs::write("certs/server.pem", server_cert.pem())?;
    fs::write("certs/server.key", server_key.serialize_pem())?;
    fs::write("certs/client.pem", client_cert.pem())?;
    fs::write("certs/client.key", client_key.serialize_pem())?;
    println!("wrote 5 files to certs/");
    Ok(())
}
```

**What happens:**

1. `CertificateParams` describes a certificate: names (SAN), the subject's distinguished name (here only the CN), validity dates, and whether it is a CA. `KeyPair::generate()` makes a new key pair (ECDSA on the P-256 curve, by default).
2. The CA is **self-signed**: `CertifiedIssuer::self_signed` signs the CA's own certificate with its own key and keeps the key, so it can sign more. `IsCa::Ca` and the `KeyCertSign` usage say "this certificate may sign other certificates".
3. `server.signed_by(&server_key, &ca)` creates the server certificate for `server_key`'s public key and signs it with the CA's key. `CertificateParams::new` turns `"127.0.0.1"` into an IP SAN and `"localhost"` into a DNS (Domain Name System) SAN by itself.
4. Short lifetimes (7 and 30 days) are deliberate: test certificates that expire soon do little harm if they leak.
5. `.pem()` and `.serialize_pem()` write the certificates and the keys as PEM text.

```bash
cargo run -q --bin certgen
openssl x509 -in certs/server.pem -noout -subject -issuer -dates -ext subjectAltName
```

```text
wrote 5 files to certs/
subject=CN=localhost
issuer=CN=p15 test CA
notBefore=Nov 18 07:36:26 2026 GMT
notAfter=Nov 25 07:36:26 2026 GMT
X509v3 Subject Alternative Name: 
    DNS:localhost, IP Address:127.0.0.1
```

Subject and issuer are now **different**: this is a real chain of two (leaf → our CA), not the self-signed leaf of Lesson 10.11.

## 3. A TLS server with tokio-rustls

`rustls` itself works on any byte stream. `tokio-rustls` connects it to Tokio: a `TlsAcceptor` takes a `TcpStream`, performs the handshake and returns a `TlsStream`, which you read and write like the plain stream. Create `src/bin/tls_server.rs`:

```rust
use std::net::SocketAddr;
use std::sync::Arc;

use rustls::pki_types::pem::PemObject;
use rustls::pki_types::{CertificateDer, PrivateKeyDer};
use rustls::server::WebPkiClientVerifier;
use rustls::{RootCertStore, ServerConfig};
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio_rustls::TlsAcceptor;
use tokio_rustls::server::TlsStream;

fn load_config(mutual: bool) -> Result<ServerConfig, Box<dyn std::error::Error>> {
    let certs =
        CertificateDer::pem_file_iter("certs/server.pem")?.collect::<Result<Vec<_>, _>>()?;
    let key = PrivateKeyDer::from_pem_file("certs/server.key")?;
    let builder = ServerConfig::builder();
    let mut config = if mutual {
        let mut roots = RootCertStore::empty();
        roots.add(CertificateDer::from_pem_file("certs/ca.pem")?)?;
        let verifier = WebPkiClientVerifier::builder(Arc::new(roots)).build()?;
        builder
            .with_client_cert_verifier(verifier)
            .with_single_cert(certs, key)?
    } else {
        builder.with_no_client_auth().with_single_cert(certs, key)?
    };
    config.alpn_protocols = vec![b"http/1.1".to_vec()];
    Ok(config)
}

async fn handle(mut tls: TlsStream<TcpStream>, peer: SocketAddr) -> std::io::Result<()> {
    let (_, conn) = tls.get_ref();
    let alpn = conn
        .alpn_protocol()
        .map(|p| String::from_utf8_lossy(p).into_owned());
    let client = conn.peer_certificates().map(|chain| chain.len());
    println!(
        "{peer}: {:?}, SNI {:?}, ALPN {:?}, client certificates: {:?}",
        conn.protocol_version().unwrap(),
        conn.server_name(),
        alpn,
        client
    );

    let mut buf = [0u8; 1024];
    let _ = tls.read(&mut buf).await?; // the request; we answer every request the same way
    let body = "hello over TLS\n";
    let response = format!(
        "HTTP/1.1 200 OK\r\ncontent-length: {}\r\nconnection: close\r\n\r\n{body}",
        body.len()
    );
    tls.write_all(response.as_bytes()).await?;
    tls.shutdown().await
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mutual = std::env::args().any(|a| a == "--mtls");
    let acceptor = TlsAcceptor::from(Arc::new(load_config(mutual)?));
    let listener = TcpListener::bind("127.0.0.1:8443").await?;
    println!("listening on https://localhost:8443 (mutual TLS: {mutual})");

    loop {
        let (tcp, peer) = listener.accept().await?;
        let acceptor = acceptor.clone();
        tokio::spawn(async move {
            match acceptor.accept(tcp).await {
                Ok(tls) => {
                    if let Err(err) = handle(tls, peer).await {
                        eprintln!("{peer}: {err}");
                    }
                }
                Err(err) => eprintln!("{peer}: handshake failed: {err}"),
            }
        });
    }
}
```

**What happens:**

1. `load_config` reads the PEM files. The trait `PemObject` (from `rustls::pki_types`) gives `pem_file_iter`, which yields every certificate in a file (a chain can have several), and `from_pem_file` for one item such as the key.
2. `ServerConfig::builder()` starts with safe defaults: TLS 1.3 and 1.2, modern cipher suites, the one installed crypto provider. `with_no_client_auth()` means "anyone may connect" (normal for a public server); `with_single_cert` sets the chain and the key and checks that they belong together. The `mutual` branch is for Section 5.
3. `alpn_protocols` lists what this server speaks. If a client offers ALPN and nothing matches, the handshake fails, which is better than speaking the wrong protocol.
4. The `ServerConfig` is wrapped in an `Arc`, because every connection shares it. `TlsAcceptor` is cheap to clone: each task gets its own handle.
5. `acceptor.accept(tcp).await` runs the whole handshake. Only if it succeeds do you get a `TlsStream`; a failed handshake is logged and only this connection ends.
6. `tls.get_ref()` returns the TCP stream and the `rustls` connection state, from which we read the TLS version, the SNI the client sent, the agreed ALPN protocol and the client certificates.
7. `tls.shutdown()` sends `close_notify` and then closes the connection. Without it, the client cannot tell a clean end from a cut connection (see "Common mistakes").

Start it in a second terminal, from the package folder (the paths `certs/…` are relative to it):

```bash
cargo run -q --bin tls_server
```

```text
listening on https://localhost:8443 (mutual TLS: false)
```

Back in the first terminal, try `curl` without and with our CA:

```bash
curl -sS https://localhost:8443/ 2>&1 | head -1
curl -sS --cacert certs/ca.pem https://localhost:8443/
curl -sv --cacert certs/ca.pem https://localhost:8443/ 2>&1 | grep -E 'SSL connection|ALPN|subject:|issuer:|verify ok'
```

```text
curl: (60) SSL certificate problem: unable to get local issuer certificate
hello over TLS
* ALPN: curl offers h2,http/1.1
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 / [blank] / UNDEF
* ALPN: server accepted http/1.1
*  subject: CN=localhost
*  issuer: CN=p15 test CA
*  SSL certificate verify ok.
```

**What happens:** without `--cacert`, `curl` uses the system trust store, where our CA is not ("unable to get local issuer certificate"). With `--cacert certs/ca.pem` the chain leads to a trusted root and the name `localhost` is in the SAN, so verification succeeds. `curl` offered `h2` and `http/1.1`; our server picked `http/1.1`. (The macOS `curl` is built with LibreSSL, which prints the cipher suite in its own way.)

The server terminal shows one line per connection:

```text
127.0.0.1:52161: handshake failed: received fatal alert: UnknownCA
127.0.0.1:52163: TLSv1_3, SNI Some("localhost"), ALPN Some("http/1.1"), client certificates: None
127.0.0.1:52165: TLSv1_3, SNI Some("localhost"), ALPN Some("http/1.1"), client certificates: None
```

The first `curl` rejected the certificate and told the server why with an **alert** (`UnknownCA`) before closing. Keep the server running.

> 💡 **Tip:** to serve the `hyper` router from Lesson 15.9 over TLS, accept the TLS stream first and then pass `TokioIo::new(tls_stream)` to `serve_connection`. `hyper` does not care whether the bytes are encrypted underneath. Set `alpn_protocols` to `h2` and `http/1.1` when you also serve HTTP/2.

## 4. A Rust client that trusts your CA

Create `src/bin/tls_client.rs`. It is the client of Lesson 10.11, but asynchronous, and with **one** trusted root: our CA.

```rust
use std::sync::Arc;

use rustls::pki_types::pem::PemObject;
use rustls::pki_types::{CertificateDer, ServerName};
use rustls::{ClientConfig, RootCertStore};
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;
use tokio_rustls::TlsConnector;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let name = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "localhost".to_string());

    // Trust exactly one root: our own CA.
    let mut roots = RootCertStore::empty();
    roots.add(CertificateDer::from_pem_file("certs/ca.pem")?)?;
    let config = ClientConfig::builder()
        .with_root_certificates(roots)
        .with_no_client_auth();
    let connector = TlsConnector::from(Arc::new(config));

    let tcp = TcpStream::connect("127.0.0.1:8443").await?;
    let server_name = ServerName::try_from(name.clone())?;
    let mut tls = connector.connect(server_name, tcp).await?;

    tls.write_all(format!("GET / HTTP/1.1\r\nHost: {name}\r\n\r\n").as_bytes())
        .await?;
    let mut reply = String::new();
    tls.read_to_string(&mut reply).await?;
    println!("{}", reply.lines().next().unwrap_or(""));
    println!("{}", reply.lines().last().unwrap_or(""));
    Ok(())
}
```

**What happens:**

1. `RootCertStore::empty()` plus one `add` means: trust nothing except our CA. This is the safe way to talk to an internal service with a private CA.
2. The program always connects to `127.0.0.1:8443`, but the **name it checks** comes from the command line. `ServerName` is both the name that must appear in the SAN and the SNI the client sends.
3. `connector.connect(name, tcp)` runs the handshake; any verification error comes back here, before a single byte of the request is sent.

```bash
cargo run -q --bin tls_client
cargo run -q --bin tls_client -- 127.0.0.1
cargo run -q --bin tls_client -- other.test
```

```text
HTTP/1.1 200 OK
hello over TLS
HTTP/1.1 200 OK
hello over TLS
Error: Custom { kind: InvalidData, error: InvalidCertificate(NotValidForNameContext { expected: DnsName("other.test"), presented: ["DnsName(\"localhost\")", "IpAddress(127.0.0.1)"] }) }
```

**What happens:** `localhost` and `127.0.0.1` are both in the SAN, so both work. For `other.test` the chain is fine but the name is not, and the error lists exactly what the certificate offers. The server log shows the difference on its side:

```text
127.0.0.1:52166: TLSv1_3, SNI Some("localhost"), ALPN None, client certificates: None
127.0.0.1:52167: TLSv1_3, SNI None, ALPN None, client certificates: None
127.0.0.1:52168: handshake failed: received fatal alert: BadCertificate
```

`SNI None` for `127.0.0.1`: SNI can only carry host names, never IP addresses. `ALPN None`: our client did not offer ALPN, which is allowed. Stop the server with `Ctrl+C`.

## 5. Mutual TLS

Normally only the server proves who it is. In **mutual TLS** the client also sends a certificate, and the server accepts only clients whose certificate is signed by a CA it trusts. Between internal services this replaces passwords and API (Application Programming Interface) keys: the certificate *is* the identity.

The `mutual` branch of `load_config` does this: `WebPkiClientVerifier::builder(roots)` builds a verifier that checks client certificates against our CA, and `with_client_cert_verifier` makes it required. Start the server with `--mtls`:

```bash
cargo run -q --bin tls_server -- --mtls
```

Then call it without and with the client certificate:

```bash
curl -sS --cacert certs/ca.pem https://localhost:8443/
curl -sS --cacert certs/ca.pem --cert certs/client.pem --key certs/client.key https://localhost:8443/
cargo run -q --bin tls_client
```

```text
curl: (56) LibreSSL SSL_read: LibreSSL/3.3.6: error:1404C45C:SSL routines:ST_OK:reason(1116), errno 0
hello over TLS
Error: Custom { kind: InvalidData, error: AlertReceived(CertificateRequired) }
```

And the server:

```text
listening on https://localhost:8443 (mutual TLS: true)
127.0.0.1:52139: handshake failed: peer sent no certificates
127.0.0.1:52141: TLSv1_3, SNI Some("localhost"), ALPN Some("http/1.1"), client certificates: Some(1)
127.0.0.1:52142: handshake failed: peer sent no certificates
```

**What happens:** without a client certificate, the server ends the handshake with the alert `CertificateRequired`. LibreSSL's message is cryptic (reason 1116 is that alert); the `rustls` client names it. With `--cert` and `--key`, `curl` proves that it owns `client-1`'s key, and the server sees one client certificate. In a real service you would now read the client's name from that certificate and decide what it may do (authorization, Lesson 10.7).

> ⚡ **Performance:** a full handshake costs one network round trip (TLS 1.3) and some public-key math on both sides; after that, encryption is cheap. Keep connections open and reuse them (keep-alive, connection pools). `rustls` also supports *session resumption*, which lets a returning client skip most of the handshake work.

## Common mistakes

**1. Two crypto providers in one program.** If any dependency enables `rustls`'s other provider (`aws-lc-rs` is the default of many crates, for example `reqwest`), `rustls` no longer knows which one to use, and `ServerConfig::builder()` panics at run time:

```text
thread 'main' (5758865) panicked at ~/.cargo/registry/src/…/rustls-0.23.46/src/crypto/mod.rs:249:14:

Could not automatically determine the process-level CryptoProvider from Rustls crate features.
Call CryptoProvider::install_default() before this point to select a provider manually, or make sure exactly one of the 'aws-lc-rs' and 'ring' features is enabled.
```

Fix: either keep exactly one provider feature in the whole dependency tree (`cargo tree -e features -i rustls` shows who enables what), or choose one explicitly at the start of `main`:

```rust
    rustls::crypto::ring::default_provider()
        .install_default()
        .expect("no other provider was installed before");
```

**2. Closing without `close_notify`.** If the server ends `handle` after `write_all` without `tls.shutdown().await`, `curl` still prints the body, but the `rustls` client refuses the data:

```text
Error: Custom { kind: UnexpectedEof, error: "peer closed connection without sending TLS close_notify: https://docs.rs/rustls/latest/rustls/manual/_03_howto/index.html#unexpected-eof" }
```

An attacker who can cut TCP connections could otherwise make a truncated response look complete. Always shut down a TLS stream when you are done writing.

**3. Speaking plain HTTP to a TLS port.** `curl http://localhost:8443/` gets `curl: (1) Received HTTP/0.9 when not allowed` (it could not understand the TLS bytes), and the server logs `handshake failed: received corrupt message of type InvalidContentType`. When you see either message, check the URL scheme.

**4. Starting the server from the wrong folder.** The paths `certs/…` are relative to the current directory. Started elsewhere, the server stops at once with `Error: Io(Os { code: 2, kind: NotFound, message: "No such file or directory" })`. Real services take the paths from configuration (Lesson 16.13) and print which file failed.

## Exercises

### Exercise 1 — One more name (easy)

Make the server certificate valid for `api.localhost` as well, regenerate the files, restart the server and check with `curl` that `api.localhost` works and `www.localhost` does not. `curl --resolve api.localhost:8443:127.0.0.1` sends that name to `127.0.0.1` for one command.

<details>
<summary>Hint</summary>

The names come from the `vec![…]` passed to `CertificateParams::new` for the server.

</details>

<details>
<summary>Answer</summary>

```rust
    let mut server = CertificateParams::new(vec![
        "localhost".to_string(),
        "api.localhost".to_string(),
        "127.0.0.1".to_string(),
    ])?;
```

```bash
cargo run -q --bin certgen
cargo run -q --bin tls_server      # in the second terminal, after Ctrl+C
curl -sS --cacert certs/ca.pem --resolve api.localhost:8443:127.0.0.1 https://api.localhost:8443/
curl -sS --cacert certs/ca.pem --resolve www.localhost:8443:127.0.0.1 https://www.localhost:8443/ 2>&1 | head -1
```

```text
wrote 5 files to certs/
hello over TLS
curl: (60) SSL: no alternative certificate subject name matches target host name 'www.localhost'
```

Why it works: the client checks its name against the SAN list. Regenerating also created a **new CA**, so every client must use the new `ca.pem`; a server still running with the old files would now fail with `BadSignature` (same CA name, different key).

</details>

### Exercise 2 — A client with a certificate (medium)

Write `src/bin/tls_client_cert.rs`: the client of Section 4, but it also presents `certs/client.pem` with `certs/client.key`. Run it against `tls_server -- --mtls` and check the server log.

<details>
<summary>Hint</summary>

Replace `.with_no_client_auth()` by `.with_client_auth_cert(chain, key)?`, where `chain` comes from `CertificateDer::pem_file_iter` and `key` from `PrivateKeyDer::from_pem_file`.

</details>

<details>
<summary>Answer</summary>

Copy `src/bin/tls_client.rs` to `src/bin/tls_client_cert.rs`. Add `PrivateKeyDer` to the `use rustls::pki_types::{…}` line and replace the creation of `config` by:

```rust
    let chain =
        CertificateDer::pem_file_iter("certs/client.pem")?.collect::<Result<Vec<_>, _>>()?;
    let key = PrivateKeyDer::from_pem_file("certs/client.key")?;
    let config = ClientConfig::builder()
        .with_root_certificates(roots)
        .with_client_auth_cert(chain, key)?;
```

With the server started as `cargo run -q --bin tls_server -- --mtls`, `cargo run -q --bin tls_client_cert` prints:

```text
HTTP/1.1 200 OK
hello over TLS
```

The server prints `127.0.0.1:52143: TLSv1_3, SNI Some("localhost"), ALPN None, client certificates: Some(1)`.

Why it works: during the handshake the server asks for a certificate; the client sends `client-1`'s certificate and proves with a signature that it has the matching private key. The server's verifier checks that our CA signed it.

</details>

### Exercise 3 — Test TLS without files (hard)

Write `tests/tls.rs` with two tests that need no files and no fixed port: generate a CA and a server certificate **in memory** with `rcgen`, start a TLS server on `127.0.0.1:0` that writes `hello` and shuts down, then (1) a client that trusts this CA receives `hello`; (2) a client that trusts a *different* CA gets an error that mentions `UnknownIssuer`.

<details>
<summary>Hint</summary>

`ca.der()` gives the CA certificate in DER (the binary form) for `RootCertStore::add`; `cert.der().clone()` gives the server certificate. The key: `PrivatePkcs8KeyDer::from(key.serialize_der()).into()`. Give the two CAs different common names; with equal names `rustls` reports `BadSignature` instead.

</details>

<details>
<summary>Answer</summary>

```rust
use std::net::SocketAddr;
use std::sync::Arc;

use rcgen::{BasicConstraints, CertificateParams, CertifiedIssuer, DnType, IsCa, KeyPair};
use rustls::pki_types::{PrivatePkcs8KeyDer, ServerName};
use rustls::{ClientConfig, RootCertStore, ServerConfig};
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio_rustls::{TlsAcceptor, TlsConnector};

/// A new test CA that lives only in memory.
fn new_ca(name: &str) -> CertifiedIssuer<'static, KeyPair> {
    let mut params = CertificateParams::new(Vec::<String>::new()).unwrap();
    params.distinguished_name.push(DnType::CommonName, name);
    params.is_ca = IsCa::Ca(BasicConstraints::Unconstrained);
    CertifiedIssuer::self_signed(params, KeyPair::generate().unwrap()).unwrap()
}

/// Starts a TLS server for "localhost" signed by `ca`; it says "hello" to every client.
async fn start_server(ca: &CertifiedIssuer<'static, KeyPair>) -> SocketAddr {
    let key = KeyPair::generate().unwrap();
    let params = CertificateParams::new(vec!["localhost".to_string()]).unwrap();
    let cert = params.signed_by(&key, ca).unwrap();
    let key_der = PrivatePkcs8KeyDer::from(key.serialize_der()).into();
    let config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(vec![cert.der().clone()], key_der)
        .unwrap();
    let acceptor = TlsAcceptor::from(Arc::new(config));

    let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    tokio::spawn(async move {
        loop {
            let (tcp, _) = listener.accept().await.unwrap();
            if let Ok(mut tls) = acceptor.accept(tcp).await {
                tls.write_all(b"hello").await.unwrap();
                tls.shutdown().await.unwrap();
            }
        }
    });
    addr
}

/// Connects to `addr` as "localhost", trusting only `ca`, and returns what the server sent.
async fn fetch(
    addr: SocketAddr,
    ca: &CertifiedIssuer<'static, KeyPair>,
) -> std::io::Result<String> {
    let mut roots = RootCertStore::empty();
    roots.add(ca.der().clone()).unwrap();
    let config = ClientConfig::builder()
        .with_root_certificates(roots)
        .with_no_client_auth();
    let connector = TlsConnector::from(Arc::new(config));
    let tcp = TcpStream::connect(addr).await?;
    let name = ServerName::try_from("localhost").unwrap();
    let mut tls = connector.connect(name, tcp).await?;
    let mut text = String::new();
    tls.read_to_string(&mut text).await?;
    Ok(text)
}

#[tokio::test]
async fn a_client_that_trusts_our_ca_gets_the_greeting() {
    let ca = new_ca("test CA");
    let addr = start_server(&ca).await;
    assert_eq!(fetch(addr, &ca).await.unwrap(), "hello");
}

#[tokio::test]
async fn a_client_that_trusts_another_ca_is_rejected() {
    let ca = new_ca("test CA");
    let addr = start_server(&ca).await;
    let err = fetch(addr, &new_ca("other CA")).await.unwrap_err();
    assert!(
        err.to_string().contains("UnknownIssuer"),
        "unexpected error: {err}"
    );
}
```

```bash
cargo test --test tls
```

```text
running 2 tests
test a_client_that_trusts_another_ca_is_rejected ... ok
test a_client_that_trusts_our_ca_gets_the_greeting ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: every test builds its own little PKI (CA, certificates, trust) in memory, so tests do not depend on files, dates on disk or each other. The negative test is as important as the positive one: it proves that the client really verifies, instead of accepting everything.

</details>

## Quiz

1. Which three things does a TLS server need, and which of them must stay secret?

<details>
<summary>Answer</summary>

A certificate chain, the matching private key, and a configuration (versions, ALPN, client authentication). Only the private key is secret; the chain is sent to every client.

</details>

2. The client connects to `127.0.0.1` and the server log shows `SNI None`. Why?

<details>
<summary>Answer</summary>

SNI can only carry a host name, not an IP address, so a client that connects by IP sends no SNI. The certificate check still works because the certificate has `127.0.0.1` as an IP SAN.

</details>

3. What does `tls.shutdown().await` send, and what goes wrong without it?

<details>
<summary>Answer</summary>

It sends the TLS `close_notify` alert, then closes the connection. Without it, a careful client (like the `rustls` one) reports `UnexpectedEof`, because it cannot distinguish the end of the data from an attacker cutting the connection.

</details>

4. In mutual TLS, what does the server check, and what does it gain?

<details>
<summary>Answer</summary>

It requires a client certificate, checks that a trusted CA signed it and that the client owns the matching key. It gains a strong, password-free identity for every client, which it can use for authorization.

</details>

5. Your program panics in `ServerConfig::builder()` with "Could not automatically determine the process-level CryptoProvider". What happened?

<details>
<summary>Answer</summary>

Two crypto providers (`ring` and `aws-lc-rs`) are enabled in the dependency tree, so `rustls` cannot pick one. Keep exactly one provider feature, or install one with `CryptoProvider::install_default()` (for example `rustls::crypto::ring::default_provider().install_default()`) at the start of `main`.

</details>

## Summary

- A TLS server needs a certificate chain, a private key that never leaves it, and a `ServerConfig`; the client needs the right trusted root and the expected name.
- For development, make your own CA with `rcgen`, sign short-lived server and client certificates, and give `ca.pem` only to your own clients.
- `TlsAcceptor::accept(tcp)` turns a `TcpStream` into a `TlsStream` after a full handshake; read SNI, ALPN and client certificates from `get_ref()`.
- A client with `RootCertStore::empty()` plus your CA trusts exactly one root; name errors list what the certificate offers.
- Mutual TLS (`WebPkiClientVerifier`) makes clients prove their identity with certificates.
- Use one crypto provider, always `shutdown()` a TLS stream, and never turn verification off.

## Sources and further reading

- [rustls documentation](https://docs.rs/rustls/latest/rustls/) — `ServerConfig`, `ClientConfig`, crypto providers and the how-to manual.
- [tokio-rustls documentation](https://docs.rs/tokio-rustls/latest/tokio_rustls/) — `TlsAcceptor` and `TlsConnector`.
- [rcgen documentation](https://docs.rs/rcgen/latest/rcgen/) — `CertificateParams`, `CertifiedIssuer` and key generation.
- [rustls-pki-types: PemObject](https://docs.rs/rustls-pki-types/latest/rustls_pki_types/pem/trait.PemObject.html) — reading certificates and keys from PEM files.
- [RFC 8446 — The Transport Layer Security (TLS) Protocol Version 1.3](https://www.rfc-editor.org/rfc/rfc8446) — the protocol itself; section 4.2 lists extensions such as SNI and ALPN.
- [Let's Encrypt — How It Works](https://letsencrypt.org/how-it-works/) — how public servers get free certificates automatically.

---

[🏠 Start](../START-HERE.md) · [← 15.11 Project: a DNS resolver](11-project-dns-resolver.md) · [📋 Phase 15](README.md) · [15.13 QUIC and HTTP/3 →](13-quic-and-http3.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
