# eMudhraPwn

**Offline X.509 certificate generator built on the compromised eMudhra intermediate CA.**

`eMudhraPwn` is a single-file, fully client-side web tool that issues X.509 v3 TLS
certificates signed by an embedded eMudhra intermediate CA. It exists as a practical
demonstration of the October 2026 eMudhra certificate compromise.

> **Authorized testing only.** The embedded CA is compromised and the private key is
> trivially weak (EC P-384 scalar `d = 4`). Updated browser and OS trust stores
> distrust this CA. Do not use generated certificates for production, public-facing
> services, impersonation, or any system you do not own or have explicit permission
> to test.

---

## Links

- **Live version:** <https://emudpwn.eleet.dev/>
- **Source code:** <https://github.com/ShotokanZH/eMudhra-pwn>

---

## Background

On 19 October 2026, Cloudflare became aware that the private keys for eight eMudhra
intermediate certificates were compromised. The keys were weak enough that an attacker
could create forged certificates for arbitrary domains. eMudhraPwn packages one of
those compromised intermediate CAs, `EM OV TLS CA - G3C` (issued by
`emSign Root TLS CA - G3`), together with its private key into a local certificate
generator.

The tool does not exploit a new vulnerability. It demonstrates the practical impact of
the known key compromise: anyone with this page can sign certificates for any domain.

Further reading:

- [Cloudflare — 8 eMudhra certificates are compromised](https://blog.cloudflare.com/emudhra-certificate-compromise/)
- [Community test site and affected package status](https://emudhra.we-have-a-problem.com/)

---

## Features

- **Single static file** — `index.html`, no build step, no CDN, no external JavaScript
  libraries, no network access during page load or certificate generation.
- **Offline by design** — the page enforces CSP `connect-src 'none'`; the CA private key
  never leaves the browser.
- **Embedded, read-only CA** — the certificate and private key are hardcoded and shown
  in the UI in read-only mode. The CA cannot be changed from the interface.
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
  certificate, individually or as a ZIP archive.
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
| `index.html` | The complete standalone application. |
| `cert.pem` | CA certificate; reference copy of the CA embedded in `index.html`. |
| `private.pem` | CA private key; reference copy of the compromised key embedded in `index.html`. |
| `README.md` | This document. |

`cert.pem` and `private.pem` are reference copies of the CA material hardcoded in
`index.html`; the page does not fetch them at runtime.

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
2. The CA section is fixed and read-only. Expand **Embedded CA** to inspect the
   certificate and private key.
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
| `<name>.fullchain.pem` | Leaf certificate followed by the intermediate CA. |
| `cert.pem` | The embedded CA certificate (inside the ZIP). |
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
- **Self-checks**: CSR and certificate signatures are verified with WebCrypto before
  the result is shown. The generated certificate is also parsed again to catch encoding
  errors.

---

## Limitations

- The CA is fixed in the HTML. To use a different CA, edit `DEFAULT_CA_CERT` and
  `DEFAULT_CA_KEY` in `index.html`.
- Only unencrypted PEM private keys are supported.
- Only end-entity certificates are generated; there is no sub-CA issuance.
- Supported CA key types: EC P-256/P-384/P-521 and RSA (PKCS#8, SEC1, PKCS#1).
  P-224, Ed25519, DSA and RSA-PSS CA keys are not supported.
- No CRL, OCSP or AIA extensions are added.
- The tool requires WebCrypto. Very old browsers and some restrictive `file://`
  configurations will not work; use a local HTTP server in that case.
- The generated certificates chain to a compromised CA and are not trusted by updated
  trust stores. They are not suitable for production or public use.

---

## Security and responsible use

- The CA private key is embedded in the HTML and available in the read-only
  **Embedded CA** section. This is inherent to the demonstration; anyone with the file
  has the key.
- The certificate is only trusted where the embedded CA (or its root chain) is trusted
  and not revoked. Installing that CA into a production trust store would undo the
  protection that the eMudhra incident response is trying to restore.
- Use the tool only in isolated labs, CTFs, or environments where you have explicit
  written authorization.
- The page does not make network requests while generating certificates. The GitHub,
  Cloudflare and Ko-fi links are external navigations and open only when clicked.
- No generated private key or CA key is transmitted or stored by the page.

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
- Built with browser WebCrypto and a hand-written ASN.1 DER encoder/parser. No
  third-party JavaScript libraries are used.

---

## Disclaimer

This project is provided for security research, education and authorized testing. The
author is not responsible for misuse, damage, or unauthorized access resulting from its
use. Use it at your own risk and only against systems you own or are explicitly
authorized to test.
