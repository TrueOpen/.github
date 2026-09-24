<p align="center">
  <a href="https://trueopen.ai">
    <img src="./assets/trueopen-lockup.png" width="420" alt="TrueOpen.ai">
  </a>
</p>

<h1 align="center">Verifiable and Private AI Inference</h1>

<p align="center">
  TrueOpen is an open GPU network for serving open-source models with
  verifiable computation and privacy protection.
</p>

<p align="center">
  <a href="https://trueopen.ai">Website</a> &middot;
  <a href="https://github.com/TrueOpen/docs/blob/main/content/assets/TrueOpen_Whitepaper.pdf">Whitepaper</a> &middot;
  <a href="https://github.com/TrueOpen/docs">Documentation</a> &middot;
  <a href="https://trueopen.ai/research.html">Research</a> &middot;
  <a href="https://trueopen.ai/roadmap.html">Roadmap</a>
</p>

---

## What we are building

An open GPU network for open-source models comes down to two problems. Everything else in the protocol exists to serve them.

| 1. Verification: validity and cost | 2. User privacy protection 🛡️ |
| --- | --- |
| Prove that remote inference used the agreed model, precision, and computation, while paying for only a fraction of the compute. | Keep who submitted a task and what it contained away from everyone who does not need it, without weakening verification or accountability. |

Open-source model weights are only the beginning. Users also need confidence that remote computation was performed as agreed, without giving up control of their identity or their data.

## 1. How verification works

Re-running a generation costs as much as the generation itself, and a full cryptographic proof of a large model is far more expensive. TrueOpen therefore uses two mechanisms that verify the *whole* answer while paying for only a *fraction* of the compute, and lets stake and penalties turn a partial detection probability into a full economic deterrent.

- **Logprob Majority-Consensus Verification (LMCV)** — three independently selected Verifiers each run one parallel teacher-forced prefill over the Worker's `input + output`, check every token's log-probability, rank and top-*k* membership (plus layer-0 expert routing for MoE models), commit their results, then reveal. Two matching results form the verdict.
- **Sampled Layerwise Proofs (SLP)** — for fixed-point quantized models, the Worker commits every layer-group boundary before random sampling; the two end chunks and a random sample of inner chunks are proven cryptographically. The verifier checks the proof without the model weights and can escalate to a full proof over the same commitments.

**Measured cost** (single runs; conditions in the linked reports):

| What | Result | Conditions |
| --- | --- | --- |
| One LMCV verification pass | ≈ 1% of the generation's GPU time (Worker/Verifier ≈ 80×–114×) | Qwen3-8B BF16, single RTX 4090, vLLM, input×output length grid; the only exception is 1-token outputs |
| Three LMCV Verifiers together | 1.9%–14.8% of one generation | same benchmark |
| LMCV model identity | Same-model BF16 replays stay in a narrow band across 6000ws / H100 / 4090 / L4; FP8, AWQ and smaller substitutes separate on p99, rank-delta and top-*k* overlap; MoE layer-0 routing reaches TPR 1.000 with zero false passes at threshold 0.060 | Qwen3-32B, Qwen3-8B, Qwen3.6-35B-A3B |
| SLP on Llama-2-70B | Seal 163 boundaries + prove 5 chunks: 1,259 s; proof 4.34 MiB; verify 46 s with no access to weights | 256-thread CPU server, 2 TB RAM, context 8, 3 sampled chunks + 2 anchors |
| SLP sampled vs. full proof | s = 5: 22% of full proving time, 6.8% of proof size, 9.6% of verification time | TinyLlama-1.1B, 47 chunks, host CPU |
| SLP batching | 12 concurrent requests in one proof: 181.9 s, ≈ 6.5× cheaper than separate proofs | TinyLlama-1.1B, slot length 16 |
| SLP detection limit | Single-audit detection = sampling coverage (1.9% at 70B with s = 3); a post-sealing beacon removes the Fiat–Shamir grinding attack | analytical + measured |

Why this is affordable: verification runs on a prefill, not a decode, so it is memory-bandwidth-cheap and parallel; only two of three Verifiers need to agree; proofs cover a sample rather than every layer; and because a Worker's stake is at risk on every audit, even a low per-audit detection probability makes cheating unprofitable over repeated tasks.

Reports and code: [LMCV paper](https://github.com/TrueOpen/lmcv-experiments/blob/main/paper/LMCV.md) · [LMCV experiments and tools](https://github.com/TrueOpen/lmcv-experiments) · [SLP paper (arXiv:2609.27367)](https://arxiv.org/abs/2609.27367) · [SLP experiment report](https://github.com/TrueOpen/slp-experiments/blob/main/REPORT.md) · [SLP raw data](https://github.com/TrueOpen/slp-experiments) · [Verification overview](https://trueopen.ai/verification.html)

## 2. How user privacy works

Privacy has two layers: hiding *who* paid for a task, and hiding *what* the task contains. Service nodes stay publicly accountable in both.

**Identity: a shielded pool and per-task keys**

- Users deposit into a shared shielded pool and hold private notes. To fund a task, the user submits a zero-knowledge proof of spending authority and value conservation without revealing which note is spent; a unique nullifier prevents double spending.
- When a task is accepted, its budget moves into escrow and change returns to the pool as new private notes; refunds at settlement return the same way, without the user coming online. Challenge bonds and verification budgets can also be paid from the pool.
- The SDK creates a fresh control key and encryption recipient key for every task, and relayers submit transactions, so tasks never share a long-term user key or an on-chain session. Private mode also routes traffic through relays so the user's network origin is not tied to the request.
- The model, budget, fees, execution status, and Worker, Verifier, and Builder identities stay public, and verification, settlement, and penalty rules are unchanged.

**Content: optional encryption with phase-based key delivery**

- Inputs and outputs are encrypted with authenticated encryption before upload. Builders store and relay only ciphertext; the project team, storage nodes, and other organizations cannot decrypt content through identity or administrative privilege.
- Keys are never given to candidates. After the Worker is chosen with future-block randomness, the SDK wraps the data key for that Worker's authenticated encryption key; outputs are encrypted and decrypted chunk by chunk as they stream.
- Verifiers receive the input, output, and token-record keys only after they are selected, and the key to the Worker's own comparison values only after they have committed, so encryption does not open a path to copying results.
- Integrity is checked without decryption: Builders verify stored and streamed bytes against ciphertext commitments, while the SDK and Verifiers check content commitments after decrypting. Worker signatures, not the AES-GCM tag, bind each streamed chunk.

**Who can see what**

| Participant | Access |
| --- | --- |
| Public | Authorization records, commitments, and settlement records; no task content |
| Builder | Ciphertext it stores and relays |
| Selected Worker | The plaintext input it needs and the output and evidence it produces |
| Selected Verifier | The data needed to verify; the Worker's comparison values only in the reveal phase |
| Authorized reviewer | Data within a dispute's scope; opening a challenge alone grants no decryption right |

**Limits we state up front.** Encryption cannot hide the input from the Worker that runs it, cannot revoke plaintext an authorized node has already read, and does not hide metadata such as data length and timing. Public deposit and withdrawal amounts and times may still offer linking clues. Users who choose encryption must keep their SDK online until verification ends to deliver keys on time. Commitments and settlement records stay on-chain permanently, and stored data is kept until its on-chain cleanup height.

## Explore TrueOpen

- [Whitepaper (PDF)](https://github.com/TrueOpen/docs/blob/main/content/assets/TrueOpen_Whitepaper.pdf)
- [Whitepaper (web)](https://github.com/TrueOpen/docs/blob/main/content/whitepaper.md)
- [Protocol documentation](https://github.com/TrueOpen/docs)
- [Verification research](https://trueopen.ai/verification.html)
- [Seal, Then Sample: Sampled Layerwise Proofs for Verifiable LLM Inference from GPT-2 to 70B (arXiv:2609.27367)](https://arxiv.org/abs/2609.27367)
- [Research and experimental reports](https://trueopen.ai/research.html)
- [End-to-end task flow](https://trueopen.ai/task-flow.html)
- [Performance and scaling](https://trueopen.ai/scaling.html)
- [Development roadmap](https://trueopen.ai/roadmap.html)

## Current status

TrueOpen is in active research, protocol design, and development.

Specifications and interfaces may evolve as verification methods, privacy mechanisms, and network coordination are tested. The shielded pool and content encryption are specified in the whitepaper and under development; the current test network runs without them. Production network access and compatible wallet integrations are not yet generally available.

## Build with us

We welcome contributors with experience in:

- AI inference and model verification
- Distributed GPU systems
- Applied cryptography and privacy
- Blockchain and consensus protocols
- SDKs, developer tooling, and infrastructure

Start with the [documentation](https://github.com/TrueOpen/docs), then open an issue to share a question or proposal.

## Security

TrueOpen will never ask for your wallet seed phrase or private key.

Use only links published on [trueopen.ai](https://trueopen.ai) and repositories under the [TrueOpen](https://github.com/TrueOpen) organization.

---

<p align="center">
  <a href="https://trueopen.ai">trueopen.ai</a> &middot;
  <a href="mailto:service@trueopen.ai">service@trueopen.ai</a>
</p>
