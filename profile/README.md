<div align="center">

# ⚡ MCB-RS (`MCBRS`)

**A modular, high-performance, clean-room Rust ecosystem for Minecraft: Bedrock Edition.**

[![Rust Edition](https://img.shields.io/badge/rust-2024_edition-orange.svg?logo=rust)](https://www.rust-lang.org/)
[![Minimum Rust Version](https://img.shields.io/badge/MSRV-1.88%2B-blue.svg?logo=rust)](https://www.rust-lang.org/)
[![License: MIT](https://img.shields.io/badge/license-MIT-purple.svg)](https://opensource.org/licenses/MIT)
[![Unsafe Code](https://img.shields.io/badge/unsafe-forbidden-success.svg)](#security--engineering-standards)

</div>

---

## 🎯 Overview

**MCB-RS** (`MCBRS`) is an open-source engineering effort building a modern, type-safe, and strictly decoupled server and networking stack for **Minecraft: Bedrock Edition** in Rust.

Rather than bundling networking, packet serialization, static game registries, and world logic into a single monolithic codebase, **MCB-RS** isolates each layer into a standalone, reusable crate with automated schema synchronization and strict wire-fidelity testing:

1. **Transport (`MCBRS-NetherNet`)** — Pure WebRTC transport, HTTP signaling, identity assertions, and SCTP channel framing.
2. **Serialization (`MCBRS-Protocol`)** — Automated code generation and wire-level binary codecs for every Bedrock packet, type, and enum.
3. **Registries (`MCBRS-id`)** — Automated extraction of canonical block states, item IDs, entity metadata, and FNV-1a network hashes.
4. **Core Server (`MCBRS`)** — The final async server runtime orchestrating compression, encryption, worlds, entities, and plugins.

---

## 📊 Ecosystem Status & Roadmap

**Overall Ecosystem Progress: `50%` (2 / 4 Core Foundations Completed)**

| Repository | Crate | Role | Status | Progress |
| :--- | :--- | :--- | :---: | :---: |
| [**`MCBRS-NetherNet`**](https://github.com/MCB-RS/MCBRS-NetherNet) | `mcbrs-nethernet` | WebRTC transport, HTTP signaling (`/v1/join`), JOSE/ECDSA identity assertions, SCTP framing | ✅ **Production Ready** | ![100%](https://progress-bar.xyz/100/?scale=100&title=Done&width=120&color=2ea44f) |
| [**`MCBRS-Protocol`**](https://github.com/MCB-RS/MCBRS-Protocol) | `mcbrs-protocol` | Automated codegen (`proto-codegen`), VarInt/NBT/primitives codec, ~300 packets & ~700 types | ✅ **Production Ready** | ![100%](https://progress-bar.xyz/100/?scale=100&title=Done&width=120&color=2ea44f) |
| [**`MCBRS-id`**](https://github.com/MCB-RS/MCBRS-id) | `mcbrs-id` | Automated codegen for blocks, items, entities, block states, and FNV-1a 32-bit runtime hashes | 🚧 **Next Up** | ![0%](https://progress-bar.xyz/0/?scale=100&title=Planned&width=120&color=6e7681) |
| [**`MCB-RS`**]((https://github.com/MCB-RS/MCB-RS)) | `mcbrs` | Dedicated server runtime (session state, Snappy/Deflate, AES-256-CFB8, worlds, ticking & plugin API) | 🗓️ **Planned** | ![0%](https://progress-bar.xyz/0/?scale=100&title=Planned&width=120&color=6e7681) |

---

## 🏗️ Architecture & Separation of Concerns

Each crate in the **MCB-RS** organization has a strict boundary and zero circular dependencies:

```text
                     ┌──────────────────────────────────────────────┐
                     │            Minecraft Bedrock Client          │
                     └──────────────────────┬───────────────────────┘
                                            │
                       HTTP Signaling (/v1/join) + WebRTC (DTLS/SCTP)
                                            │
                                            ▼
┌───────────────────────────────────────────────────────────────────────────────────────┐
│ 1. MCBRS-NetherNet (100%)                                                             │
│    • HTTP Signaling (`GET /v1/join`, `POST /v1/join/{networkId}`)                     │
│    • Client & Server `a=identity` Validation (ES256/ES384 JWT + Detached JWS + TOFU)  │
│    • Full-ICE Negotiation (Host, 1:1 NAT `srflx`, STUN/TURN, Candidate Inference)     │
│    • SCTP `ReliableDataChannel` & `UnreliableDataChannel` + 1-byte Countdown Framing  │
└───────────────────────────────────────────┬───────────────────────────────────────────┘
                                            │ Raw Reassembled Payloads (`Bytes`)
                                            ▼
┌───────────────────────────────────────────────────────────────────────────────────────┐
│ 4. MCBRS Server Runtime (0% - Planned)                                                │
│    • Session Lifecycle, Batch Compression (Snappy / Deflate) & AES-256 Encryption     │
│    • Pre-Compressed Broadcasts, World Management, Entity Ticking & Plugin Hooks       │
└───────────────────┬───────────────────────────────────────────────┬───────────────────┘
                    │ Framed Packet Batches                         │ Block / Item Lookups
                    ▼                                               ▼
┌───────────────────────────────────────────┐   ┌───────────────────────────────────────┐
│ 2. MCBRS-Protocol (100%)                  │   │ 3. MCBRS-id (0% - Next Up)            │
│    • VarUInt32 Batch & Packet Framing     │   │    • Canonical Block & Item Enums     │
│    • Complete Generated Packet AST        │   │    • Block State Permutations         │
│    • Network NBT, Primitives & Validation │   │    • FNV-1a Runtime ID Hashes         │
└───────────────────────────────────────────┘   └───────────────────────────────────────┘
```

---

## 📦 Core Repositories

### 1. [`MCBRS-NetherNet`](https://github.com/MCB-RS/MCBRS-NetherNet) — `100% Complete`
Complete server and client implementation of Minecraft Bedrock's **NetherNet HTTP Signaling** and WebRTC transport layer, verified end-to-end against the real Minecraft Bedrock client:
- **HTTP Signaling Endpoints**: `GET /v1/join` (`ServerInfo` capability check) and `POST /v1/join/{networkId}` (full-ICE SDP offer/answer exchange) powered by `axum` and `tokio`.
- **Cryptographic Identity & Trust**: Full `a=identity` parsing and generation (`ES256` / `ES384` JWKs and DER SPKI, RFC 8785 canonical JSON, RFC 7515 detached JWS over DTLS `a=fingerprint` lines, and RFC 7638 thumbprints for TOFU pinning).
- **WebRTC & NAT Traversal**: Built on `webrtc-rs` (`0.21`) with `MaxBundle`, 1:1 NAT `public_ips`, optional STUN/TURN `ice_servers`, client candidate inference behind NAT, and reverse-proxy IP extraction (`trusted_proxies`).
- **Hardened Data Channels**: Transparent fragmentation and countdown reassembly (`N..0`) on `ReliableDataChannel`, single-frame enforcement on `UnreliableDataChannel`, and strict byte-budget semaphores preventing memory exhaustion.

### 2. [`MCBRS-Protocol`](https://github.com/MCB-RS/MCBRS-Protocol) — `100% Complete`
Automated, wire-accurate serialization library covering the entire Minecraft Bedrock network protocol:
- **Automated Code Generation (`proto-codegen`)**: Parses official JSON protocol schemas into strongly-typed Rust structs, enums, and `oneOf` variants, breaking recursive type cycles automatically.
- **Wire-Fidelity Engine**: Handles Cereal schema nuances accurately (`std::optional` vs. default-valued fields, indexed vs. const-tagged `oneOf` branches, raw binary `BedrockString` payloads, Big-Endian handshake fields, and `ConnectionRequest` JWT chains).
- **Zero-Touch Maintenance**: Scheduled GitHub Actions workflows track upstream protocol updates and open automated pull requests without redistributing proprietary schema files.
- **Built-in Constraint Validation**: Every generated packet and type implements `BedrockCodec` and `Validate` (enforcing schema `maxLength`, `minItems`/`maxItems`, and numeric bounds).

### 3. `MCBRS-id` — `0% (In Preparation)`
Upcoming static game registry crate generated automatically from upstream Bedrock data definitions:
- Type-safe `BlockId`, `ItemId`, `EntityId`, and `BiomeId` enums.
- Canonical block state property permutations and precomputed FNV-1a 32-bit network runtime hashes.
- Zero runtime parsing overhead via compile-time tables.

### 4. `MCB-RS` — `0% (Planned)`
The final modular server software combining `mcbrs-nethernet`, `mcbrs-protocol`, and `mcbrs-id`.

---

## 🛡️ Security & Engineering Standards

- **Memory & DoS Safety**: `#![forbid(unsafe_code)]` across transport crates, strict byte budgets on incoming/outgoing queues, bounded VarInt/string/array readers, and timeout-guarded handshakes.
- **Clean-Room & EULA-Compliant Workflow**: No proprietary Mojang files or assets are committed or redistributed in `MCB-RS` repositories or published crates; upstream schemas are fetched locally on demand during code generation.
- **Continuous Verification**: Enforced by CI via `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`, and `RUSTDOCFLAGS="-D warnings" cargo doc`.

---

## ⚖️ Licensing & Legal Disclaimer

All original source code across the **MCB-RS** organization is licensed under the [**MIT License**](https://opensource.org/licenses/MIT).

No files, schemas, or assets owned by Mojang AB or Microsoft Corporation are distributed in our repositories or published crates. Protocol schemas and data definitions fetched locally on demand during code generation remain `(c) Mojang AB` and are subject to their respective licenses and the [Minecraft End User License Agreement (EULA)](https://www.minecraft.net/en-us/eula).

**NOT AN OFFICIAL MINECRAFT PRODUCT. NOT APPROVED BY OR ASSOCIATED WITH MOJANG OR MICROSOFT.**

- **Publisher / Owner:** MCB-RS (Enzo Zizzi) — solely responsible for all software published under this organization. Mojang AB and Microsoft Corporation are not responsible for these products or services.
- **Contact:** For legal, security, or technical inquiries, please open an issue on the relevant repository in the [MCB-RS GitHub Organization](https://github.com/MCB-RS) or contact the organization owner directly via the email listed on the GitHub profile.
