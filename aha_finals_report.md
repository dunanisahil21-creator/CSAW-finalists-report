# CSAW 2026 AI Hardware Attack Challenge — Finalist Review

**Sahil Dunani** · Net ID sd6429 · based on the finalist presentations, 13 Nov 2026

A review of two of the eight finalist teams — how each used LLMs, how their Trojans work (with the
five-axis taxonomy), and where they could improve. Both targeted the Calico *Hackster* board
(RP2040 + Lattice iCE40-UP5K FPGA), recovering each design from its bitstream before inserting
Trojans.

## Taxonomy at a glance — qualifier key-leak Trojan

| Axis | Ithaca's Fall | GOONIES |
|---|---|---|
| **Insertion phase** | Design / RTL (recovered netlist) | Design / RTL (recovered netlist) |
| **Abstraction level** | Gate / RTL (RTLIL edits) | Gate / RTL |
| **Activation** | Single magic MOSI frame `0x13579BDF` before first START | Two-block sequence `0xDEADC0DE` → `0xFEEDFACE` (~2⁻⁶⁴) |
| **Payload** | Information leakage — key recovery | Information leakage — key exfiltration |
| **Location** | SPI result / readout path | Crypto result / readout path |

Across their other challenges both teams also reach integrity-corruption and denial payloads (C2
output corruption, C3 mute / bypass) in the DSP datapath; GOONIES additionally used a
cryptographic-weakening payload (nonce reuse).

## Team 1 — Ithaca's Fall

Bill Nguyen · Buster Coden · Jaden Wijata — framework open-sourced as **IkaCore** (`github.com/Squid-Proxy-Lovers/IkaCore`)

**How they used LLMs.** They built a reusable, Apache-2.0 multi-agent framework and ran the whole
attack through it rather than prompting a chat box.

- IkaCore provides `IkaBaseAgent`, `IkaStage` (bounded staged execution) and `IkaWorkflow` (a
  multi-agent graph), six provider backends (OpenAI, Anthropic, DeepSeek, Gemini, OpenRouter,
  Codex), a memory module, checkpoint/resume, human-in-the-loop, and a test suite with CI.
- Pipeline: **Recon → Understand → Strategy → Insert → Exploit → Verify**, logged stage by stage
  (Attack Architecture → Insertion Search → Candidate Screen → Insertion Refinement → Exploit
  Generation → **Verification Tournament** → Packaging).
- Independent "attack-architect" agents each emit a strict **JSON attack contract** — trigger as
  Boolean / temporal rules, payload as internal-transform + external-observation, non-empty
  negative cases — under anti-hallucination constraints ("no key claim unless `cipher.key` matches";
  "no key recovery from a fault without algebraic proof").
- Insertion happens at the **RTLIL / gate level** (recipes carry a canonical graph hash and
  `add_cell` ops) — the "AST-manipulation" tier of the rubric. Models: GPT-Sol 5.6/6/6.1, DeepSeek
  V4.1, GLM 5.3; they rejected Claude and Opus on context and cost.

**How the Trojans work.**

- **C1 — SPI cipher.** `0x13579BDF` before the first START makes MISO leak a key-dependent word
  (`0xEE026691`); a custom decryption then recovers the key `0x8E043CB1`.
- **C2 — ASCON-AEAD128.** A transparent interposer, armed by reserved opcodes: key/secret exfil
  (`0xE7`, replays the I²C key-load image on MISO), secret leak (`0xB2`), output corruption (`0xC3`).
- **C3 — FIR audio.** LSB-steganography message, filter bypass, and mute — each gated by an
  8-sample password; dormant output matches stock.

**Where they could improve.**

- The C1 leak surfaces an anomalous word on MISO and fires on a single fixed frame; a
  multi-stage / encoded trigger and a quieter exfil would resist fuzzing and entropy checks.
- They relied on "Daybreak Blue" / non-refusing models to clear guardrails (and listed refusals as
  a problem) — a reproducibility and legitimacy risk, since their own prompt is already tightly
  scoped and authorized.
- No synthesis-overhead (LUT / FF) numbers or detector results were shown; quantifying footprint and
  running a GNN detector would finish the stealth case.

## Team 2 — GOONIES

Jessica Ryn · Kevin Wu · Kenneth Lei · Kyson Wu (UNLV / UCLA) — "one board, three targets, six Trojans"

**How they used LLMs.** Also agentic, but the opposite design choice: a generate → insert → verify
→ retry loop with a **bespoke agent setup per challenge** rather than one framework.

- **C1** — Claude drives the design loop from the chat UI while a Python pipeline automates
  insert → simulate → verify (no API key; reruns reproduce the identical Trojan).
- **C2** — an offline six-model agent pipeline (retrieve → verify → generate).
- **C3** — eight agents: three reverse-engineer the filter, four design competing Trojans, one logs
  everything ("Project Scribe"). Agents drive the OSS CAD Suite (Yosys, Icarus Verilog, IceStorm,
  nextpnr).

**How the Trojans work.**

- **C1 — SPI cipher.** A two-block "knock": `0xDEADC0DE` arms, `0xFEEDFACE` fires (~2⁻⁶⁴); on firing,
  MISO returns the key `0x8E043CB1`. 72/72 benign transactions are bit-identical.
- **C2 — ASCON.** Three Trojans — leak, bypass, and **nonce reuse** (weakens the cipher without
  changing the I/O shape).
- **C3 — FIR audio.** A 128-frame filter bypass, demonstrated on the board.

**Where they could improve.**

- The knock pairs a superb trigger with a loud payload (the raw key on MISO); encoding or throttling
  the leak would help.
- Per-challenge bespoke swarms are great for a two-day contest but harder to reuse or audit than a
  single framework like IkaCore.
- Model refusals and log sprawl were flagged; consolidating models and adding a detector-in-the-loop
  would strengthen reproducibility and stealth. Their nonce-reuse payload is the most elegant of the
  six and deserves a detectability analysis.

---

*Sources: finalist presentations, CSAW 2026 AI Hardware Attack Challenge (NYU), 13 Nov 2026;
Ithaca's Fall IkaCore repository (github.com/Squid-Proxy-Lovers/IkaCore).*
