  GNU nano 7.2                      mitm-hsts.md                                
---
title: "Hardware Security & Fault Injection: Building an Automated Emulator-Based Framework in Rust (Part 1)"
description: ""
pubDate: 'Jul 31 2026'
lang: en
heroImage: '../../assets/fault_inject_p1.jpeg'

---

# Hardware Security & Fault Injection: Building an Automated Emulator-Based Framework in Rust (Part 1)

When we think about software security, we usually assume that instructions executed by the CPU are deterministic and untamperable. If the code says `mov r0, #0`, the register will become `0`. But in the realm of **Hardware Security**, physical attacks like voltage glitching, clock manipulation, or electromagnetic pulses can disrupt a processor's normal operation—causing it to **skip critical instructions** entirely.

Testing these physical vulnerabilities usually requires an expensive hardware lab with oscilloscopes, pulse generators, and target chips. I wanted to understand how these instruction-skip attacks work without spending thousands of dollars on physical gear. So I built an **emulator-based fault injection framework in Rust** using the Unicorn Engine.

This is a write-up of how I set up the environment, automated the attack campaign, and what the execution trace revealed.

---

## The Setup

My setup consisted of:

* A Rust project utilizing the `unicorn-engine` crate.
* An emulated ARM architecture running in little-endian mode.
* A small ARM assembly snippet mapped into virtual memory at `0x1000`.

The assembly logic was deceptively simple:

```rust
let code = [
    0x00, 0x00, 0xa0, 0xe3, // 0x1000: mov r0, #0 (Default: Access Denied)
    0x01, 0x00, 0xa0, 0xe3, // 0x1004: mov r0, #1 (Grant Access)
];

```

The intended behavior: `R0` represents authorization state ($0 = \text{Denied}$, $1 = \text{Granted}$). Under normal conditions, execution flows sequentially, starting with access denied and ending with granted access. The goal was to simulate a hardware fault that skips the first instruction and forces a bypass.

---

## How Software-Based Fault Interception Works

To simulate a hardware glitch without physical equipment, I used **Code Hooks**.

A Hook acts as an observer attached to the emulated CPU core. Whenever the Program Counter (`PC`) reaches a specified address, the Hook triggers a closure before the instruction executes. To simulate a glitch that causes an instruction to be skipped, the Hook intercepts the target address and manually advances the Program Counter past the instruction length ($4 \text{ bytes}$ for ARM32):

```rust
if let Some(target_addr) = addr {
    emu.add_code_hook(0x1000, 0x4000, move |emu, address, size| {
        if address == target_addr {
            println!("[!] Fault injected: Skipping address {:#x}", address);
            // Advance PC past current 4-byte instruction
            emu.reg_write(RegisterARM::PC, address + size as u64).unwrap();
        }
    }).expect("Failed to add code hook");
}

```

---

## Automating the Campaign

Testing single addresses manually is inefficient for real-world binaries. I structured the framework into two components:

1. **`run_simulation(addr: Option<u64>) -> u64`**: Sets up a fresh emulator instance, writes memory, conditionally registers the fault hook for `addr`, runs the execution, and returns the final value of register `R0`.
2. **Campaign Loop (`main`)**: Executes a baseline run (no fault injected), then iterates through every candidate instruction address, invoking the simulation with `Some(target_address)`.

```rust
fn main() {
    println!("=== 1. Baseline Run (No Fault) ===");
    let r0_base = run_simulation(None);
    println!("Baseline Result: R0 = {}\n", r0_base);

    println!("=== 2. Automatic Fault Injection Campaign ===");
    let targets = [0x1000, 0x1004];

    for target in targets {
        println!("\n[Testing Skip on Address {:#x}]", target);
        let r0_fault = run_simulation(Some(target));

        if r0_fault == 1 {
            println!("==> [VULNERABLE] Skipping {:#x} allowed BYPASS! (R0 = 1)", target);
        } else {
            println!("==> [SAFE] Skipping {:#x} kept system secure. (R0 = {})", target, r0_fault);
        }
    }
}

```

---

## What I Saw

Running `cargo run` gave the following terminal trace:

```text
=== 1. Baseline Run (No Fault) ===
Enter R1 value: 222
Baseline Result: R0 = 1

=== 2. Automatic Fault Injection Campaign ===

[Testing Skip on Address 0x1000]
Tracing at address 0x1000, size: 4
[!] Fault injected: Skipping address 0x1000
Tracing at address 0x1004, size: 4
==> [VULNERABLE] Address 0x1000 allowed BYPASS! (R0 = 1)

[Testing Skip on Address 0x1004]
Tracing at address 0x1000, size: 4
Tracing at address 0x1004, size: 4
[!] Fault injected: Skipping address 0x1004
==> [SAFE] Skipping 0x1004 kept system secure. (R0 = 0)

```

### Analyzing the Trace

* **When `0x1000` was skipped:** The instruction `mov r0, #0` was bypassed. The CPU jumped directly to `0x1004` (`mov r0, #1`). The system evaluated $R0 = 1$ and reported **`[VULNERABLE]`**.
* **When `0x1004` was skipped:** The CPU executed `0x1000` normally, setting $R0 = 0$. The hook then skipped `0x1004`. The register remained $0$, meaning the authorization grant failed, and the system reported **`[SAFE]`**.

---

## What I Took Away from This

1. **Hardware fault injection can be effectively modeled in software:** You don't need a high-end lab setup to start understanding how instruction skipping alters control flow.
2. **Default states matter:** Single points of failure in assembly initialization make binaries extremely fragile against single-glitch attacks.
3. **Automation is key:** An automated framework makes scanning binaries for vulnerable instructions fast and repeatable.

In [Part 2](https://itsmehrnaz.github.io/blog/), I explore how to defend against this exact vulnerability by implementing **Double Verification** in assembly and auditing the hardened binary with this framework.
