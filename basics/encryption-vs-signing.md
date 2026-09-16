---
title: Encryption vs. Signing
description: How encryption, digital signatures, and shared-key authentication solve different security problems.
---

# Encryption vs. Signing

**Encryption keeps data secret. Signing lets someone check who produced data and
whether it changed.** They solve different problems, so a system may need both.

| Question | Encryption | Signing |
| --- | --- | --- |
| Main goal | Can an unauthorized person read this? | Did this come from a trusted sender, unchanged? |
| What a recipient does | Decrypts ciphertext to recover the message | Verifies a signature against the message |
| Does it hide the message? | Yes | No |
| Familiar use | Protecting data in transit or at rest | Verifying a token, webhook, or software release |

## Encryption: keep the contents private

Suppose Alice sends Bob a private document. She encrypts it; someone who
intercepts the ciphertext cannot read the document without the right key. Bob
decrypts it with that key.

Most data encryption uses a **symmetric key**: the same secret key encrypts and
decrypts. **Asymmetric encryption** uses a public key to encrypt and its matching
private key to decrypt. In practice, systems often use public-key techniques to
agree on a short-lived symmetric key, then use that key for the bulk data.

Encryption by itself does not necessarily prove that ciphertext was not changed.
Use an **authenticated encryption** scheme, such as AES-GCM, when you need both
secrecy and an integrity check. HTTPS/TLS uses authenticated encryption for
application traffic.

## Signing: prove origin and integrity

Suppose Alice publishes a document that anyone may read, but Bob needs to know
it really came from her. Alice signs the document with her **private key**. Bob
uses Alice's **public key** to verify the signature. If the document changes,
verification fails. The document remains readable to everyone; signing does
not encrypt it.

```text
Alice: document + private key -> signature
Bob:   document + signature + Alice's public key -> valid or invalid
```

Verification is meaningful only if Bob can trust that the public key belongs
to Alice. A valid signature also does **not** mean a message is fresh: a copied
old request may still verify. Use an expiration time, timestamp, or nonce when
replay matters.

### Shared-secret signing: HMAC

Some systems use an **HMAC** instead of a public/private-key digital signature.
The sender and verifier share one secret key, which they use to create and
check a message authentication code. This detects changes and proves the sender
knew the shared secret, but either party with that secret could have generated
the code. An HMAC does not hide the message either.

## In a system design

- **HTTPS:** TLS authenticates the server and protects traffic with encryption
  and integrity checks.
- **Signed JWT or webhook:** The receiver verifies the signature or HMAC before
  trusting the claims or acting on the event; the payload is not secret merely
  because it is signed.
- **Private file:** Encrypt the file if its contents must stay confidential;
  sign or authenticate it if recipients also need to detect tampering.

## Further reading

- [NIST: Digital signature](https://csrc.nist.gov/glossary/term/digital_signature)
- [OWASP: Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [RFC 8446: TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446.html)
