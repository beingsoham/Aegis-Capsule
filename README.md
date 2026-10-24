# Aegis — Project Capsule

**A post-quantum secure communications overlay.**
Team **QUBIT** · Problem Statement **SIH26237** — *Post Quantum Secure Communication Overlay* · Theme: Cybersecurity · Institution: Vishwakarma University · Event: Smart India Hackathon 2026

> Prototype. Not audited. Not for production use.

This capsule is a single self-contained snapshot of the Aegis project: what it is, what it claims, what it explicitly does *not* claim, and what has actually been built so far (the marketing/landing site in this repo). Every technical claim below traces back to a section (`§n`) of `Aegis_System_Documentation_Manual.docx`, the project's source of truth.

---

## 1. What Aegis Is

Aegis is a cryptographic communications overlay that keeps one hybrid post-quantum session alive across multiple transports — including a direct local-network path when no infrastructure is reachable — while the relay server stays structurally blind to message content.

Two endpoints establish and maintain a quantum-resistant encrypted session whose security does not depend on any particular network path, and whose relay infrastructure never possesses the material required to read the messages it carries. (§1)

### Why Aegis exists (§2)

| # | Problem | Description |
|---|---|---|
| 01 | **Harvest now, decrypt later** | A passive attacker records encrypted traffic today and decrypts it retroactively once quantum hardware exists — already a real threat for government, defense, medical and legal communication. (§2.1) |
| 02 | **Network dependency** | Almost every secure messenger assumes a reachable central server. In disaster zones, rural regions and temporary command posts that path is frequently unavailable, even when two devices sit on the same local network. (§2.2) |
| 03 | **Server trust** | Even a reachable server is a single point of compromise — hackable, misconfigurable, legally compellable. (§2.3) |

> Aegis treats these as one design problem, not three features bolted together. (§2.4)

### Honest positioning (§3.1)

**We did not invent post-quantum messaging.** Signal has shipped PQXDH in production. PQCChat offers browser-based post-quantum messaging with ML-KEM and ML-DSA. Aegis does not compete with those products on chat features, UX, or ecosystem maturity — it would lose that comparison. It targets the problem sitting underneath all of them: every one assumes a central server is reachable.

### The claim, in three independently demonstrable parts (§3.2)

1. **Hybrid post-quantum security** — the same well-studied primitives other products use, composed carefully. (§6)
2. **Demonstrated transport independence** — the identical cryptographic session and packet format running over more than one transport, with the crypto engine unaware which is active. (§11)
3. **A blind relay** — verifiable by inspecting server storage and finding no plaintext or session secrets. (§14)

---

## 2. The Cryptographic Stack (§5)

**Rule: never invent a cryptographic primitive.** Every algorithm below is a standardized, publicly reviewed construction — the engineering work is in composing them correctly, not designing new mathematics.

| # | Algorithm | Standard | Role | Purpose |
|---|---|---|---|---|
| 01 | `ML-KEM-768` | FIPS 203 | The quantum half | Lattice-based (Module-LWE) key establishment; does not encrypt the message itself. |
| 02 | `X25519` | RFC 7748 | The classical half | Elliptic-curve Diffie-Hellman — the classical half of the hybrid handshake. |
| 03 | `ML-DSA-65` | FIPS 204 | Who the key belongs to | Signs identity and pre-key bundles so a recipient can verify a public key's owner. |
| 04 | `HKDF-SHA-256` | RFC 5869 | Two secrets, one key | Combines the classical and post-quantum secrets into a root key and derives session sub-keys. |
| 05 | `AES-256-GCM` | SP 800-38D | Confidentiality and integrity | Authenticated encryption of the message payload. |
| 06 | `Double Ratchet` | — | The key never sits still | Continuously replaces message keys and periodically re-runs a DH step to recover after compromise. |

> "Combining two algorithms automatically makes the system secure." It does not, by itself.

### The hybrid handshake (§6)

```
root_key = HKDF-SHA-256( SS_X25519 ‖ SS_MLKEM , context )
```

The shared secret is never transmitted. ML-KEM produces a ciphertext that only the holder of the matching private key can decapsulate — intercepting the ciphertext does not reveal the secret.

Four requirements make the composition trustworthy, not just the algorithm choice:

1. **Fixed concatenation order** — both sides concatenate secrets identically; a mismatch fails the session closed rather than silently.
2. **Context binding in HKDF** — the `info` parameter binds the key to protocol version, negotiated algorithm IDs, and both sides' identity.
3. **Downgrade resistance** — the negotiated algorithm set is covered by the authenticated transcript, so stripping the post-quantum component is detectable.
4. **Key confirmation** — both sides derive and compare a short confirmation value before treating the session as live.

> Key establishment is not authentication. ML-KEM guarantees that two parties who complete the protocol correctly arrive at the same secret. It says nothing about whether the party on the other end is who they claim to be.

---

## 3. Transport Independence (§11)

> It calls `send(ciphertext)`. It never calls `send_over_wifi()`.

The crypto engine produces one thing — an encrypted packet — and hands it to a transport abstraction without knowing or caring which concrete transport is underneath. This is the architectural property the project is actually trying to prove, demonstrated rather than just diagrammed:

1. Alice and Bob are chatting through the internet relay.
2. The router the relay depends on fails; Alice's transport manager detects it (connection error / heartbeat timeout).
3. JmDNS resolves Bob's presence and address on the local network.
4. A direct Netty TCP connection is established to Bob.
5. The next outgoing packet — same ratchet state, same header format, same ciphertext structure — is sent over that connection instead. No renegotiation, no new handshake.

**Scope note:** detecting relay failure, discovering the peer and switching transports automatically is application/transport-manager logic that must be built and tested — it is not automatic just because the crypto layer is transport-agnostic. What the architecture guarantees is that the ratchet sequence and cryptographic state remain valid across the switch.

### Three named conditions (§13)

| Mode | Condition | Behavior |
|---|---|---|
| **A — Connected** | Normal internet path available | Client → relay → client |
| **B — Locally connected** | No internet, peers share a local network | Direct LAN path carries the same encrypted session |
| **C — Temporarily disconnected** | Neither internet nor shared LAN reachable | Messages encrypted and queued locally; delivery resumes once any transport returns |

> The encryption step is identical in all three. Only delivery changes.

---

## 4. The Blind Relay (§14)

The relay is designed so none of the information required to decrypt a message ever needs to reach it — a structural guarantee, verifiable by inspection, not a policy promise.

| The server **may** hold | The server **must never** hold |
|---|---|
| Account and user identifiers | Plaintext message content |
| Public identity keys and signatures | Private identity keys |
| Public pre-key bundles (X25519, ML-KEM) | Root keys |
| Encrypted message packets | Chain keys |
| Temporary offline-delivery queues | Message keys |
| Connection and session metadata | Any ratchet state |

**Demonstrated (§14.2):** inspect the backend database directly — it shows account records, public keys, signatures and ciphertext, and it can be shown live that no field yields plaintext, private keys or session secrets under any query.

**Why not just TLS? (§14.4):** TLS secures the transport connection between a client and the server — it does not stop the server reading application content, because the server is one endpoint of that connection. Aegis encrypts at the application layer, end-to-end, so the packet the relay forwards is already ciphertext it cannot open.

### The caveat we lead with (§14.3)

> "The server sees nothing" is false, and we do not claim it.

Even a correctly built blind relay still observes metadata: which accounts are communicating with which, message timing, packet frequency, approximate packet sizes, and connection information. Content confidentiality and metadata privacy are two different properties — Aegis does not currently claim metadata protection.

---

## 5. Threat Model (§15)

Nine adversary classes, each with the property expected to limit them:

| Adversary | Capability | What limits them |
|---|---|---|
| Passive network observer | Observe traffic in transit | AES-256-GCM confidentiality |
| Active network attacker | Modify, inject, drop, replay packets | AEAD auth tags + replay protection |
| Malicious relay server | Inspect/manipulate relayed traffic | Blind-relay design |
| Compromised relay (at rest) | Full read access to server storage | Storage holds only ciphertext + metadata |
| Malicious LAN peer | Present on the same local network | JmDNS gives no trust; cryptographic identity verification required |
| **Compromised endpoint (active)** | Full control of a live client | **Not mitigated while active** — stated openly |
| Stolen device (at rest) | Physical possession, powered off/locked | Encrypted-at-rest storage, OS-backed key protection |
| Long-term passive recorder | Stores ciphertext today for future decryption | The post-quantum half of the hybrid handshake |
| Future quantum adversary | Would break X25519 via Shor's algorithm | ML-KEM remains secure independent of that break |

### What Aegis explicitly does **not** defend against (§15.2)

- An endpoint that is actively compromised, while the compromise is ongoing.
- Metadata correlation by a party observing traffic patterns over time.
- Social-engineering or out-of-band compromise of the initial trust anchor, if a user never performs out-of-band verification.
- Denial of service against relay availability (mitigated *in effect* by the LAN fallback, but not a cryptographic guarantee).

---

## 6. What We Do Not Claim (§3.3)

- Certified or integrated with a specific military radio, waveform, or classified network.
- A solved military key-management system (credential issuance, revocation, escrow, classification-domain separation).
- "Unhackable", "quantum-proof", or "100% secure" — not technically meaningful claims.
- Ready for military deployment — the realistic path runs through security review, protocol audit, pen testing, procurement: a multi-year institutional process.

---

## 7. Known Gaps (§19)

> "We plan to" is never rewritten as "we have implemented." "Estimated" is never rewritten as "measured." "Should be secure" is never rewritten as "is proven secure."

| Item | Status |
|---|---|
| TOFU + fingerprint/QR identity verification | Design decided; implementation and UX not yet built. |
| Encrypted-at-rest ratchet state, atomic persistence | Design decided; crash-safety testing not complete. |
| Automatic, seamless transport handover | Crypto compatibility architected; failure-detection/switching logic still needs building and stress-testing. |
| One-time pre-key exhaustion / replenishment | Fallback specified, not implemented end-to-end. |
| Handshake timeout / resumable transfer | No timeout/fallback protocol implemented yet. |
| Additional bearer beyond TCP and LAN | Future scope. |
| **Formal protocol verification / independent audit** | **Not performed** — required before any claim stronger than "prototype." |

---

## 8. System Architecture (§4)

One session, two independent transports. The same cryptographic session runs over the relay or over a direct local connection; the crypto engine does not know which is carrying it, and the relay holds nothing that could open either.

- **Client devices A and B** each run a crypto engine (private keys never leave the device) alongside an encrypted local message store.
- **LAN transport**: JmDNS peer discovery + a direct Netty TCP channel, no internet required.
- **Spring Boot server** acts purely as a blind relay — auth service, pre-key directory, ciphertext relay, offline queue.
- **Postgres** backing store holds only public keys and ciphertext blobs.

*(See `architecture.png` / `public/architecture.png` in the landing-page repo for the diagram referenced throughout the site.)*

---

## 9. The Landing Page — What's Been Built So Far

This repo (`Landing-Page`) is the marketing/demonstration site for Aegis, built for Smart India Hackathon 2026 judging. It is a **static Next.js 15 site**, and every line of copy on it cites back to the documentation manual above.

### Stack

| | |
|---|---|
| Framework | Next.js 15, App Router, static export (`output: 'export'`) |
| Styling | CSS Modules driven by a central `styles/tokens.css` design-token file |
| 3D | three.js — two lazily-loaded canvases (hero CRT object, cryptographic-stack object grid) |
| Type | Archivo (display) + IBM Plex Mono (standards/mono) via `next/font` |
| Design system | Documented end-to-end in `DESIGN.md` — a near-black (`#101010`) canvas with one signal-yellow (`#FFD600`) accent, visual language referenced from buttermax.net's *system* only (no assets reused) |

The WebGL layer is gated off entirely under `prefers-reduced-motion`, Save-Data, low device memory, or no WebGL2 — the page is fully complete and readable without it.

### Build log (chronological, oldest → newest)

1. **`chore: scaffold Next.js 15 app with static export`** — project skeleton, static export config.
2. **`feat(design): add cinematic design tokens`** — the full token system from `DESIGN.md` (colors, type scale, spacing, motion curve) landed as CSS variables.
3. **`feat(content): add copy layer keyed to the documentation manual`** — `content/aegis.ts` created: every string of copy on the site, each one tagged with its `§n` citation back to the manual.
4. **`feat(ui): add section primitives and type system`** — shared layout/typography components for the sixteen content blocks.
5. **`feat(gl): add CRT hero object`** — the hero's cathode-ray-tube 3D object: cursor-reactive, screen cycling ciphertext ⇄ plaintext (`SGVsbG8gV29ybGQh` ⇄ `Hello World!`).
6. **`feat(gl): render the six primitives on one shared canvas`** — the cryptographic-stack object grid (§5): one shared WebGL canvas rendering all six algorithm tiles.
7. **`feat: add transport handover diagram`** — the pinned, scroll-scrubbed five-step transport-handover sequence for §11 (the project's centerpiece section).
8. **`feat(fx): add decrypt, count-up and scroll reveal`** — supporting motion effects: text-decrypt reveal, numeric count-ups, scroll-triggered tile reveals.
9. **`feat: compose the landing page`** — all sixteen blocks assembled into the final page (`app/page.tsx`).
10. **`docs: add design specification and reference renders`** — `DESIGN.md` finalized; reference screenshots captured into `docs/` (`hero.png`, `stack.png`, `full-page.png`).
11. **`chore: keep the standalone CRT tuning harness`** — `crt-prototype.html` retained as a standalone tool for tuning the CRT shader's live uniforms outside the Next.js build.
12. **`feat: add system architecture section and team details`** — the §4 architecture section and SIH team/problem-statement block added to the page and footer.
13. **`docs: add architecture section render`** — reference screenshot of the architecture section captured (`docs/architecture-section.png`).
14. **`feat: serve the documentation manual and link it from the footer`** — the full 22-section, 43-page `Aegis_System_Documentation_Manual.docx` is served as `public/aegis-documentation.pdf` and linked from the footer, so every citation on the page can be checked against its source.

### Current repo layout

```
app/          routes, layout, global styles
components/   sections, header, effects
gl/           three.js scenes — crt.ts (hero), tiles.ts (stack objects)
content/      all page copy, keyed by manual section (content/aegis.ts)
styles/       design tokens and CSS modules
docs/         reference renders (hero, stack, architecture-section, footer, full-page)
public/       architecture.png, aegis-documentation.pdf
crt-prototype.html   standalone CRT shader-tuning harness
DESIGN.md     full design specification (tokens, motion rules, copy rules, page blueprint)
README.md    project overview and run instructions
```

### Page blueprint status

All sixteen blocks specified in `DESIGN.md` are implemented and composed on the live page:

`Preloader → Header → Hero (§1) → Three problems (§2) → Honest positioning (§3.1) → Three claims (§3.2) → Cryptographic stack (§5) → Hybrid handshake (§6) → Transport independence (§11, pinned) → Three modes (§13) → Blind relay (§14.1) → Metadata caveat (§14.3) → Threat model (§15) → Not claiming (§3.3) → Known gaps (§19) → Judge FAQ (§20) → Footer (§22)`

### Open items on the design system itself (noted in `DESIGN.md`)

- Accent color (`#FFD600`, inherited from the buttermax.net reference) is flagged as unresolved for a defense-adjacent subject — a cold cyan or acid green is proposed as an alternative that would keep the system intact.
- Display typeface not finally chosen (Archivo/Outfit as free candidates).
- Seven of the nine 3D objects used across the page are not yet sourced (CC0 BoomBox and CRT candidates are settled).
- Tier-3 (no-WebGL) static-image fallbacks for the 3D tiles do not exist yet and need to be rendered once objects are final.

### Status

**Prototype. Not audited. Not for production use.** — this applies to both the Aegis protocol/architecture itself and this landing page as its demonstration surface.

---

*Capsule generated from the Aegis landing-page repository on 2026-10-24. Source files: `README.md`, `DESIGN.md`, `content/aegis.ts`, and git commit history.*
