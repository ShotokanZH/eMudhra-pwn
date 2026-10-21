# eMudhraPwn

**Offline X.509 certificate generator built on the revoked eMudhra intermediate CAs.**

`eMudhraPwn` is a single-file, fully client-side web tool that issues X.509 v3 TLS
certificates signed by any of the affected eMudhra intermediate CAs. It exists as a
practical demonstration of the October 2026 eMudhra certificate compromise.

> **Authorized testing only.** The embedded CAs are compromised and revoked
> (`caCompromise`), and their private keys are weak small integers (for example,
> `d = 4` and `d = 156`). Updated browser and OS trust stores distrust these CAs.
> Do not use generated certificates for production, public-facing services,
> impersonation, or any system you do not own or have explicit permission to test.

---

## Links

- **Live version:** <https://emudpwn.eleet.dev/>
- **Source code:** <https://github.com/ShotokanZH/eMudhra-pwn>

---

## Background

On 19 October 2026, Cloudflare became aware that the private keys for eight eMudhra
intermediate certificates were compromised. The keys were weak enough that an attacker
could create forged certificates for arbitrary domains. eMudhraPwn packages all 16
affected intermediate certificates (8 CA keys with cross-signed variants), together
with their private keys, into a local certificate generator.

The tool does not exploit a new vulnerability. It demonstrates the practical impact of
the known key compromise: anyone with this page can sign certificates with any of the
embedded CAs. All affected intermediates were revoked with reason `caCompromise`;
certificates issued by them should be rejected by updated revocation-checking clients.

Further reading:

- [Cloudflare — 8 eMudhra certificates are compromised](https://blog.cloudflare.com/emudhra-certificate-compromise/)
- [Community test site and affected package status](https://emudhra.we-have-a-problem.com/)

---

## Features

- **Single static file** — `index.html`, no build step, no CDN, no external JavaScript
  libraries, no network access during page load or certificate generation.
- **Offline by design** — the page enforces CSP `connect-src 'none'`; the CA private
  keys never leave the browser.
- **Embedded CA profiles** — all 16 affected certificates (8 keys with cross-signed
  variants) are embedded. A profile selector switches the active CA; the selected
  certificate and private key are shown read-only and cannot be edited in the UI.
- **SAN input** — multiple DNS names and IP addresses:
  - wildcard DNS names such as `*.example.com`
  - IPv4 and IPv6 addresses
  - IDN domains converted to punycode
  - duplicate removal and validation
- **Site key algorithms**:
  - RSA 2048, RSA 3072, RSA 4096
  - ECDSA P-256, ECDSA P-384
- **Private key formats**:
  - PKCS#8 (`BEGIN PRIVATE KEY`) by default
  - legacy PKCS#1 (`BEGIN RSA PRIVATE KEY`) for RSA
  - SEC1 (`BEGIN EC PRIVATE KEY`) for EC keys
- **X.509 v3 certificate** with:
  - `subjectAltName` for all requested names
  - `basicConstraints = critical, CA:FALSE`
  - `keyUsage = critical, digitalSignature[, keyEncipherment]`
  - `extendedKeyUsage = serverAuth[, clientAuth]`
  - Subject Key Identifier and Authority Key Identifier
  - random 128-bit serial number
  - issuer encoded byte-for-byte from the CA subject
  - configurable validity and subject fields (`CN`, `O`, `OU`, `C`, `ST`, `L`)
- **PKCS#10 CSR** with SAN, basic constraints, key usage and extended key usage.
- **Downloadable artifacts**: certificate, private key, CSR, full chain and CA
  chain, individually or as a ZIP archive.
- **Automatic issuer chain completion** — for the embedded profiles, `cert.pem` and
  `fullchain.pem` include the required issuer intermediate (for example,
  `emSign Root TLS CA - G3` for the emSign TLS profiles). No manual chain assembly is
  needed.
- **Internal verification** — generated CSR and certificate signatures are verified in
  the browser before the result is displayed.
- **Certificate details and OpenSSL commands** — the result panel shows the subject,
  issuer, SANs, key type, signature algorithm, validity, serial and equivalent
  verification commands.
- **UI modes** — glassmorphism/hacker theme with matrix rain, responsive layout, and
  reduced-motion support.
- **SEO and social metadata** — meta description, robots, canonical URL, Open Graph
  (`og:url` points to the live version), Twitter card and `SoftwareApplication` JSON-LD.
  Add an absolute `og:image` and `twitter:image` for rich social previews.

---

## Files

| File | Purpose |
| --- | --- |
| `index.html` | The complete standalone application, including the 16 CA profiles. |
| `cert.pem` | Default CA certificate reference (`EM OV TLS CA - G3C`, emSign chain). |
| `private.pem` | Default CA private key reference (same profile). |
| `README.md` | This document. |

`cert.pem` and `private.pem` are the reference for the default profile only. The 16
profiles are embedded in `index.html` and are based on the public Knight1/emudhra key
set. The page does not fetch these files at runtime. When you generate with a
non-default profile, the ZIP contains that profile's CA chain as `cert.pem`; the
repository `cert.pem` remains the default reference.

---

## Quick start

1. Open the hosted version at <https://emudpwn.eleet.dev/>, or open `index.html` in a
   modern browser (Chrome, Firefox, Edge, Safari).
   - WebCrypto is available from `file://` in most modern browsers.
   - If your browser blocks WebCrypto on `file://`, serve the folder locally:
     ```bash
     python3 -m http.server 8000
     ```
     then open <http://localhost:8000/>. This is still fully local.
2. Choose a CA from the **CA profile** selector. The selected material is read-only;
   expand **Embedded CA** to inspect the certificate and private key.
3. Enter one or more domains or IP addresses in **Domains and IP addresses**.
   Examples:
   ```
   example.com, *.example.com, 10.0.0.5, 2001:db8::1
   ```
4. Choose the site key type, validity, and optional subject fields.
5. Press **GENERATE CERTIFICATE**.
6. Inspect the result, copy individual files, or use **Download all (.zip)**.

---

## Outputs

| File | Description |
| --- | --- |
| `<name>.crt` | Leaf certificate in PEM format. |
| `<name>.key` | Site private key (PKCS#8 by default; PKCS#1/SEC1 optional). |
| `<name>.csr` | PKCS#10 certificate signing request. |
| `<name>.fullchain.pem` | Leaf certificate followed by the complete intermediate chain. |
| `cert.pem` | CA chain: selected CA plus its issuer intermediates (inside the ZIP). |
| `<name>-certificates.zip` | All of the above. |

The default file name is derived from the first SAN. You can override it in
**File name**.

---

## Verification

The embedded CA is an intermediate, not a root. Treat it as the trust anchor with
`-partial_chain`:

```bash
openssl verify -partial_chain -CAfile cert.pem example.com.crt
```

Inspect the certificate and CSR:

```bash
openssl x509 -in example.com.crt -noout -text
openssl req -in example.com.csr -noout -text -verify
```

Check that the certificate and private key match:

```bash
openssl x509 -in example.com.crt -noout -pubkey > cert.pub
openssl pkey -in example.com.key -pubout > key.pub
diff cert.pub key.pub
```

Run a local TLS test server:

```bash
openssl s_server -accept 4433 -cert example.com.crt -key example.com.key -www
```

Remember that public clients will reject the certificate unless they trust the
compromised CA or verification is explicitly disabled for an isolated lab.

---

## Supported inputs and defaults

### SAN entries

- DNS names: `example.com`, `*.example.com`, single-label names such as `localhost`
- IPv4: `192.0.2.10`
- IPv6: `2001:db8::1`, `::1`
- Input may be comma-separated or whitespace/newline-separated.
- Full URLs are accepted; only the host name is used.
- Optional ports are ignored for the SAN.
- IDN domains are converted to punycode before encoding.
- Duplicate SANs are removed.
- Only DNS and IP SANs are supported; email and URI SANs are not.

### Key and certificate parameters

| Parameter | Default | Notes |
| --- | --- | --- |
| Site key | RSA 2048 | RSA 2048/3072/4096 or ECDSA P-256/P-384 |
| Validity | 365 days | Range 1–3975 days |
| `notBefore` | now − 1 hour | Small backdate for clock skew |
| Serial | random 128-bit | Positive DER INTEGER |
| `CN` | first SAN | Maximum 64 characters |
| Country | empty | Must be a 2-letter country code if set |
| `serverAuth` | always | Required for TLS server use |
| `clientAuth` | optional | Enable for mutual TLS |
| Private key format | PKCS#8 | PKCS#1/SEC1 selectable |

---

## How it works

- **ASN.1 DER** is encoded and parsed directly in JavaScript.
- **WebCrypto** is used for key generation, RSA/ECDSA signing and SHA-1 digests.
- **SKI/AKI**: the Subject Key Identifier is the SHA-1 of the site public key bits; the
  Authority Key Identifier uses the CA's SKI.
- **ECDSA signatures**: WebCrypto returns raw `r || s`; the tool converts them to the
  ASN.1 DER `SEQUENCE { INTEGER r, INTEGER s }` required by X.509.
- **RSA**: PKCS#1 v1.5 with SHA-256.
- **CA key import**: SEC1 and PKCS#1 keys are wrapped in PKCS#8 for WebCrypto import.
- **CA key validation**: the tool signs a random challenge with the CA private key and
  verifies it with the CA public key, ensuring the pair matches.
- **Certificate assembly**: the TBS certificate contains the exact CA subject as issuer,
  the site SPKI, validity, subject and extensions. The TBS is signed with the CA key and
  the signature algorithm is selected from the CA key type:
  - EC P-256 → `ecdsa-with-SHA256`
  - EC P-384 → `ecdsa-with-SHA384`
  - EC P-521 → `ecdsa-with-SHA512`
  - RSA → `sha256WithRSAEncryption`
- **Chain completion**: for embedded profiles, the ZIP's `cert.pem` contains the
  selected CA plus its issuer intermediate(s); `fullchain.pem` is leaf + that chain.
  AAA profiles need no extra intermediate; emSign TLS/EV profiles add
  `emSign Root TLS CA - G3`, and emSign CS profiles add `emSign Root CS CA - G3`.
- **Self-checks**: CSR and certificate signatures are verified with WebCrypto before
  the result is shown. The generated certificate is also parsed again to catch encoding
  errors.

---

## Limitations

- The embedded CA list is fixed. To add a custom CA, edit `CA_PROFILES` in
  `index.html`.
- Only unencrypted PEM private keys are supported.
- Only end-entity certificates are generated; there is no sub-CA issuance.
- Supported CA key types: EC P-256/P-384/P-521 and RSA (PKCS#8, SEC1, PKCS#1).
  P-224, Ed25519, DSA and RSA-PSS CA keys are not supported.
- No CRL, OCSP or AIA extensions are added.
- The tool requires WebCrypto. Very old browsers and some restrictive `file://`
  configurations will not work; use a local HTTP server in that case.
- The generated certificates chain to revoked CAs and are not trusted by updated
  trust stores. They are not suitable for production or public use.

---

## Security and responsible use

- The CA private keys are embedded in the HTML and available in the read-only
  **Embedded CA** section. This is inherent to the demonstration; anyone with the file
  has the keys. All affected CAs were revoked with reason `caCompromise`.
- A certificate is only trusted where its embedded CA (or root chain) is trusted and
  not revoked. Installing these CAs into a production trust store would undo the
  protection that the eMudhra incident response is trying to restore.
- Use the tool only in isolated labs, CTFs, or environments where you have explicit
  written authorization.
- The page does not make network requests while generating certificates. The GitHub,
  Cloudflare and Ko-fi links are external navigations and open only when clicked.
- No generated private key or CA key material is transmitted or stored by the page.

---

## Credits

- **ShotokanZH** — author.
- **GitHub** — source repository:
  [ShotokanZH/eMudhra-pwn](https://github.com/ShotokanZH/eMudhra-pwn).
- **Cloudflare** — incident analysis:
  [8 eMudhra certificates are compromised](https://blog.cloudflare.com/emudhra-certificate-compromise/).
- **eMudhra** — the compromised intermediate CA and certificate data.
- **Fluepke** — reference gist for the compromised CA certificate and key material:
  [Reference gist](https://gist.github.com/Fluepke/9a103c7f990c3cc2fb5002f7476f2f48).
- **Knight1** — multi-CA Go PoC and key set used for the embedded profiles:
  [Knight1/emudhra](https://github.com/Knight1/emudhra).
- **galadran/we-have-a-problem.com** — public test chains and issuer/root data used to
  complete the embedded CA chains:
  [galadran/we-have-a-problem.com](https://github.com/galadran/we-have-a-problem.com).
- Built with browser WebCrypto and a hand-written ASN.1 DER encoder/parser. No
  third-party JavaScript libraries are used.

---

## Disclaimer

This project is provided for security research, education and authorized testing. The
author is not responsible for misuse, damage, or unauthorized access resulting from its
use. Use it at your own risk and only against systems you own or are explicitly
authorized to test.
