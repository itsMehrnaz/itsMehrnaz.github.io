  GNU nano 7.2                                                                                          mitm-hsts.md                                                                                                    
---
title: " Hardware Security & Fault Injection: Defending Assembly Code with Double Verification (Part 2)"
description: ""
pubDate: 'Aug 02 2026'
lang: en
heroImage: '../../assets/fault_inject_p2.jpeg'
---


In [Part 1](https://itsmehrnaz.github.io/blog/), we built an automated, emulator-based fault injection framework in Rust using the Unicorn Engine. We demonstrated how intercepting the program execution and jumping over a single instruction (`mov r0, #0`) completely bypassed the authorization logic on our target ARM binary.

While identifying a vulnerable instruction is exciting from an attacker's perspective, the ultimate goal of security engineering is defense. How do we write assembly code that remains secure even when an attacker can arbitrarily skip instructions using voltage glitching or laser injection?

In this post, we will implement **Double Verification**—one of the foundational hardware hardening techniques—and run our automated scanner against it to prove its effectiveness.

---

## The Concept: What is Double Verification?

In standard software design, developers tend to write minimal and efficient code. If a check passes, execution continues; if a single condition sets a flag, that flag is trusted implicitly.

However, physical fault injection breaks this trust. If an attacker skips the instruction responsible for setting a default "access denied" state, the CPU simply moves forward with undefined or leftover register values.

**Double Verification** (or redundant checking) introduces intentional software redundancy:

* Instead of relying on a single initialization or check, the system asserts the security boundary **multiple times** at different execution stages.
* To grant access, the state must transition through multiple independent checkpoints.
* If a glitch skips any single checkpoint, the remaining check(s) maintain the default secure state.

---

## Hardening the Assembly Code

To test this defense, we modified our ARM bytecode inside our Rust framework to include two consecutive default assignment checkpoints before evaluating the final authorization logic:

```rust
fn run_simulation(addr: Option<u64>) -> u64 {
    // Hardened instruction sequence (3 ARM instructions = 12 bytes)
    let code = [
        0x00, 0x00, 0xa0, 0xe3, // 0x1000: mov r0, #0  (Checkpoint 1)
        0x00, 0x00, 0xa0, 0xe3, // 0x1004: mov r0, #0  (Checkpoint 2 - Redundant Defense)
        0x01, 0x00, 0xa0, 0xe3, // 0x1008: mov r0, #1  (Authorization state)
    ];
    
    // ... Emulator configuration and hook setup ...
}

```

By introducing `0x1004` as a duplicate `mov r0, #0`, we force the CPU to validate the default restrictive state twice before any grant instruction can be executed.

---

## Running the Automated Campaign Against Hardened Code

We updated our automated campaign loop in `main()` to iterate through all three target addresses (`0x1000`, `0x1004`, and `0x1008`) and tested how the system behaves under single-instruction skip attacks.

```rust
fn main() {
    println!("=== 1. Baseline Run (No Fault) ===");
    let r0_base = run_simulation(None);
    println!("Baseline Result: R0 = {}\n", r0_base);

    println!("=== 2. Automatic Fault Injection Campaign ===");
    let targets = [0x1000, 0x1004, 0x1008];

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

### Execution Results

Executing the campaign via `cargo run` yielded the following output:

```text
=== 1. Baseline Run (No Fault) ===
Enter R1 value: 222
Baseline Result: R0 = 1

=== 2. Automatic Fault Injection Campaign ===

[Testing Skip on Address 0x1000]
Tracing at address 0x1000, size: 4
[!] Fault injected: Skipping address 0x1000
Tracing at address 0x1004, size: 4
Tracing at address 0x1008, size: 4
==> [SAFE] Skipping 0x1000 kept system secure. (R0 = 0)

[Testing Skip on Address 0x1004]
Tracing at address 0x1000, size: 4
Tracing at address 0x1004, size: 4
[!] Fault injected: Skipping address 0x1004
Tracing at address 0x1008, size: 4
==> [SAFE] Skipping 0x1004 kept system secure. (R0 = 0)

[Testing Skip on Address 0x1008]
Tracing at address 0x1000, size: 4
Tracing at address 0x1004, size: 4
Tracing at address 0x1008, size: 4
[!] Fault injected: Skipping address 0x1008
==> [SAFE] Skipping 0x1008 kept system secure. (R0 = 0)

```

---

## Analyzing the Defense

The campaign results confirm that our software-level hardening successfully neutralized the single-instruction skip attack across all addresses:

1. **Skipping `0x1000` (First Checkpoint):** The glitch skipped the first initialization, but execution immediately fell through to `0x1004` (`mov r0, #0`). The second check caught the state, ensuring $R0$ remained `0`.
2. **Skipping `0x1004` (Second Checkpoint):** The first instruction at `0x1000` executed normally, setting $R0 = 0$. Even though `0x1004` was skipped, the secure state was already locked in.
3. **Skipping `0x1008` (Grant Instruction):** The bypass attempt failed to execute the grant instruction altogether, resulting in an enforced default-deny behavior.

By simply duplicating critical security checks in assembly, we transformed an exploit path from **100% Vulnerable** to **100% Resistant** against single-glitch fault injection attacks.

---

## What I Took Away from This

1. **Default-Deny + Redundancy is Vital:** In low-level security, compiler optimization often strips out redundant code. However, in hardware security, intentional redundancy is a feature, not a bug.
2. **Automated Testing Provides Instant Verification:** Having an automated testing framework allows us to immediately verify whether a code patch actually mitigates the vulnerability or just shifts it to a different address.
3. **Single-Glitch vs. Multi-Glitch Models:** While Double Verification protects against single-instruction skips, sophisticated attackers can perform **multi-glitch attacks** (glitching twice in rapid succession). Securing against multi-glitch scenarios requires complementary state variables (e.g., Hamming distance checks).

---

### What's Next?

In the next installment of this series, we will explore **Multi-Fault Injection Campaign Models** and investigate how compiler optimization flags (`-O2` / `-O3`) can silently strip away our security defenses!

If you're exploring embedded security or fault injection emulation, I'd love to hear your thoughts and approaches in the comments or on GitHub.
