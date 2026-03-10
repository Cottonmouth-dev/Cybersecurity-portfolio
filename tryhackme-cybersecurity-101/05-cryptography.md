# Cryptography in Practice: Securing Data at Rest and in Transit

> **Module:** TryHackMe Cybersecurity 101 — Cryptography  
> **Date completed:** March 2026  
> **Author:** Vilhelm Stjernström

---

## Why Cryptography Matters in Security Operations

In a modern SOC, nearly all network traffic you analyze is encrypted. Every time a SOC analyst investigates a packet capture (PCAP), they need to understand what's encrypted and what's in plaintext. Without a solid grasp of cryptography, it's impossible to distinguish between legitimate TLS traffic from a browser and a malicious C2 (Command and Control) channel using encryption to conceal data exfiltration. Understanding cryptography isn't about cracking codes all day, it's about knowing how attackers abuse trust systems and how we can use cryptographic evidence (like hashes) to verify and investigate attacks.

---

## Core Concepts

### Symmetric vs. Asymmetric Encryption

Cryptography is a constant game of trade-offs — primarily between speed and security.

**Symmetric encryption** uses the same key to both lock and unlock data. Think of a padlock where both you and the recipient hold a copy of the exact same physical key. Algorithms like AES are extremely fast, making them perfect for encrypting large volumes of bulk data. The problem? How do you share that key securely over an untrusted network without someone eavesdropping?

**Asymmetric encryption** solves the key distribution problem. Here, everyone has a key pair: a public key (like an open padlock anyone can snap shut) and a private key (the only thing that can open it again). Algorithms like RSA are brilliant for this, but they're mathematically heavy and significantly slower.

**The real-world solution:** In practice, we use both. Protocols like TLS use asymmetric encryption first to securely agree on a shared secret (key exchange), then both parties switch to symmetric encryption to transmit the actual data quickly and efficiently. This hybrid approach gives us the best of both worlds — the security of asymmetric for key exchange and the speed of symmetric for bulk data transfer.

### Hashing — One-Way Functions

The simplest way I think about hashing: you can turn a cow into minced meat, but you can never turn the minced meat back into a cow. That's a one-way function.

Unlike encryption, which is designed to be reversible (unlocked), hashing is a one-way mathematical function. You can feed an entire book into a hash function like SHA-256 and get a fixed-length string of characters, but you can never take that string and reconstruct the book.

A good hash function exhibits the **avalanche effect** — change a single comma in the original file, and the resulting hash becomes completely different. This property is what makes hashing reliable for integrity verification.

**Common algorithms and their status:**

- **MD5** — considered broken. Researchers have demonstrated practical collision attacks, meaning two different files can produce the same hash. Never rely on MD5 for security-critical verification.
- **SHA-1** — deprecated. Collision attacks have been demonstrated (Google's SHAttered project in 2017). Still seen in legacy systems but should not be trusted.
- **SHA-256** — the current standard. Part of the SHA-2 family, no known practical attacks. This is what you'll use in SOC work daily.

**Password storage and salting:** Hashing plays a critical role in how systems store passwords. A well-designed system never stores your actual password — it stores the hash. When you log in, the system hashes your input and compares it to the stored hash. However, hashing alone isn't enough. If two users have the same password, they'll have the same hash, making the system vulnerable to **rainbow table attacks** (precomputed tables mapping common passwords to their hashes).

This is where **salting** comes in. A salt is a unique, random string appended to each password before hashing. Even if two users both choose "password123", their stored hashes will be completely different because each one was hashed with a different salt. As a SOC analyst, understanding this matters when investigating credential breaches — if a leaked database contains unsalted hashes, the impact is dramatically worse than one with properly salted hashes, because attackers can crack unsalted hashes in bulk using rainbow tables.

As a future SOC analyst, hashes would be one of my most-used tools for integrity verification. If I were to see a suspicious file in an EDR alert, my first action would be to extract its SHA-256 hash and run it against threat intelligence databases such as VirusTotal. This gives a near-instant answer on whether the file is known malware — without ever needing to open or execute the file myself. Of course, this has a clear limitation — attackers routinely modify even a single byte of a malicious file to produce a completely different hash, effectively bypassing signature-based detection. This is why hash lookups are a fast first step, not the full picture. Modern SOC environments combine hash-based detection with behavioural analysis from EDR tools that monitor what a file actually does when executed, regardless of its hash value.

### Public Key Infrastructure (PKI) and Digital Certificates

If asymmetric encryption is built on sharing public keys, a major problem arises: how do I know that the key claiming to belong to "mybank.com" actually does, and isn't controlled by an attacker sitting in a coffee shop between me and the bank (a man-in-the-middle attack)?

Think of it this way: if I lock a chest using my private key, anyone who can open it with my public key knows for sure that I'm the one who locked it — because only I have that private key. This is essentially how digital signatures work. A CA "signs" a certificate with its own private key, and your browser verifies that signature using the CA's public key. If the signature checks out, you know the certificate is legitimate.

To formalize this trust at scale, we use PKI and digital certificates. A certificate binds a public key to a specific identity (such as a domain), and this binding is vouched for by a trusted third party called a **Certificate Authority (CA)**. The trust flows in a chain: Root CA → Intermediate CA → End-entity certificate. Your browser comes pre-loaded with a set of trusted Root CAs, and any certificate that chains back to one of those roots is considered trustworthy.

**What's inside a certificate:** the domain's public key, the issuer (which CA signed it), the validity period (not before / not after dates), the digital signature from the CA, and the Subject Alternative Names (SANs) listing which domains it covers.

In a SOC environment, certificates are excellent **Indicators of Compromise (IoCs)**. An attacker setting up a phishing page or a C2 server rarely bothers to obtain a legitimate certificate from a real CA. Hunting for **self-signed certificates** in outbound network traffic heading toward external, unknown IP addresses is a classic and effective threat hunting technique. Other red flags include recently issued certificates (created days before an attack), certificates with unusual or misspelled domain names, and expired certificates still being actively used.

### TLS/SSL — How Secure Communication Actually Works

TLS is the backbone of internet security and combines all the concepts above. In a modern TLS 1.3 handshake, the following occurs:

1. **Client Hello** — the client sends its supported cipher suites and a random value
2. **Server Hello** — the server responds with its chosen cipher suite and presents its PKI certificate to prove its identity
3. **Key Exchange** — both parties use asymmetric encryption to agree on a shared secret
4. **Session Key Established** — a symmetric session key is derived from the shared secret
5. **Encrypted Communication Begins** — all subsequent data flows over fast symmetric encryption

TLS 1.3 improved on TLS 1.2 by reducing the handshake to a single round trip, removing support for insecure legacy cipher suites, and making **Perfect Forward Secrecy** mandatory — meaning even if a server's private key is compromised in the future, past recorded sessions cannot be decrypted.

For a SOC analyst, TLS is a double-edged sword. It protects user data, but it makes us blind to what's actually being sent over the network. To work around this without breaking encryption, we use techniques like **JA3/JA3S fingerprinting**. By analyzing exactly how a client sets up its TLS handshake — which cipher suites it supports, their order, the extensions used — we can create a fingerprint. Even though we can't read the data, the JA3 fingerprint can tell us: "This isn't Chrome talking — this is Metasploit Meterpreter."

---

## Hands-On: What I Practiced

### Observing the Avalanche Effect

I used `sha256sum` to generate file hashes and clearly observe how a single character change produces a completely different output:

```bash
$ echo "Hello World" | sha256sum
a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e

$ echo "Hello World!" | sha256sum
7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069
```

Adding a single exclamation mark completely transformed the hash, practically confirming why hashing is so reliable for detecting whether a file has been tampered with during transfer or storage.

### Inspecting Digital Certificates

I practiced extracting and reading digital certificates directly from the terminal using OpenSSL:

```bash
$ openssl s_client -connect example.com:443
$ openssl x509 -in cert.pem -text -noout
```

This allowed me to inspect the issuer, validity period, public key algorithm, and Subject Alternative Names — all fields that a SOC analyst would examine when investigating suspicious TLS connections.

### Cracking Hashes and Protected Files with John the Ripper

To see why salting matters in practice, I used John the Ripper to crack Windows NTLM hashes. NTLM is notoriously weak because it stores password hashes without salting — meaning identical passwords always produce identical hashes, making them vulnerable to rainbow table attacks and brute-force tools.

Beyond basic hash cracking, I also worked with John's conversion utilities — `zip2john` and `rar2john` — which extract password hashes from encrypted archive files so John can attempt to crack them. In a SOC context, this is relevant when investigating suspicious email attachments: a password-protected ZIP file attached to a phishing email might need to be cracked open for analysis during an investigation.

I also used `unshadow` to combine `/etc/passwd` and `/etc/shadow` on Linux into a format John can process — demonstrating how an attacker with access to both files could attempt to crack user passwords offline. Understanding this attack path helps me recognize what to look for when investigating a Linux system compromise.

Seeing how fast John cracked unsalted NTLM hashes was a wake-up call — what took seconds with NTLM would be exponentially harder against properly salted hashes, since precomputed tables become useless.

---

## SOC Analyst Relevance — How I'd Use This on the Job

### 1. Malware Analysis and Triage

When an alert fires for a suspicious file on an endpoint, my first instinct isn't to run the file — it's to hash it. By taking the SHA-256 value and querying threat intelligence platforms like VirusTotal, I can determine within a minute whether the file is known malware. If the hash matches a known malicious sample, I escalate directly to incident handling without risking execution.

### 2. Threat Hunting in Encrypted Traffic

Modern attacks — ransomware C2 beaconing, data exfiltration, credential theft — increasingly happen over HTTPS/TLS. I can't inspect the payload, so I use my understanding of PKI and TLS to hunt for anomalies instead: Was the certificate issued recently? Is it self-signed on traffic heading outbound to the internet? Does the JA3 fingerprint match known malicious software signatures in our threat intelligence feeds?

### 3. Incident Response and Chain of Custody

When collecting forensic evidence — such as a memory dump from an infected server — I need to guarantee that the evidence hasn't been altered. Immediately after acquisition, I generate a SHA-256 hash of the evidence file and document it formally. This ensures both the legal and technical integrity of the investigation. If the hash changes at any point, the evidence may be considered compromised.

---

## Key Takeaways

- **The performance-security balance:** Understanding why TLS uses both symmetric and asymmetric encryption gave me a deeper appreciation for how network protocols are designed to be both secure and fast — and why these design decisions directly affect what a SOC analyst can and cannot see.

- **Encryption protects, hashing verifies:** The clear distinction that encryption is two-way (for confidentiality) while hashing is one-way (for integrity) is central to knowing which tool solves which problem. They complement each other but serve fundamentally different purposes.

- **Salting prevents bulk attacks:** Learning how salts make each password hash unique — even for identical passwords — clarified why breach severity varies so dramatically depending on how passwords were stored.

- **The analyst's blind spots:** Knowing how TLS hides data forces me as an analyst to get better at analyzing network metadata (certificates, handshake parameters, JA3 fingerprints) rather than relying on reading plaintext. The encryption isn't the enemy — it's a constraint that shapes how I investigate.

---

## What I Want to Explore Further

- **JA3/JA3S Fingerprinting:** I want to dive deeper into building custom SIEM detection rules based on TLS fingerprints to proactively identify and block C2 traffic before it causes damage.

- **Ransomware Encryption Mechanisms:** How do modern ransomware groups combine symmetric and asymmetric encryption to lock systems as fast as possible, and how can you identify that process in memory before it's too late?

- **Certificate Transparency Logs:** How can SOC analysts use public CT logs to detect when a suspicious certificate has been issued for a domain their organization owns — enabling early detection of phishing infrastructure?

---

*Completed as part of TryHackMe's Cybersecurity 101 path. This write-up represents my personal understanding — no TryHackMe answers or flags are disclosed.*
