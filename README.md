HARP v12 — Production-Grade Embedded Firmware

IEC 61508 / ISO 13482 Compliant · STM32H7 Target · RTIC 2.x

HARP (Human-Assistive Robotics Platform) is a safety-critical, no-std Rust firmware stack for multi-DOF robotic systems operating in direct contact with humans. Version 12 closes four systematic engineering gaps identified in v11, achieving deterministic telemetry, non-blocking IPC, flash-endurance-safe audit logging, and latency-aware watchdog supervision.

Table of Contents

Overview

Architecture

Module Reference

Hardware Requirements

Software Requirements

Safety Certifications & Standards Compliance

Requirements Traceability Matrix

Formal Verification

What HARP v12 Solves

Key Novelties in v12

Potential Applications

v11 → v12 Improvement Summary

v12 → v13 Roadmap

Build & Verification

Overview

HARP v12 is a no_std, bare-metal Rust firmware targeting the STM32H7 microcontroller family. It is designed for robotic systems with up to 26 degrees of freedom (DOF) that operate in physically intimate contact with human subjects — such as exoskeletons, rehabilitation robots, and assistive care devices.

The firmware is structured around the RTIC 2.x real-time interrupt-driven concurrency framework and enforces strict safety gates, tamper-evident audit logging, consent-gated motion, and redundant sensor validation at every control tick.

Architecture

High-Level Diagram

┌─────────────────────────────────────────────────────────────────┐ │ RTIC Application │ │ │ │ ┌──────────────────────┐ ┌──────────────────────────────┐ │ │ │ fast_control_loop │ │ sensor_acquisition │ │ │ │ Priority 10 · 1 kHz │ │ Priority 5 · 500 Hz │ │ │ │ │ │ │ │ │ │ SecureIpc (DMA+CRC) │ │ ESKF predict + covariance │ │ │ │ JitterWatchdog │ │ bound check │ │ │ │ SafetyEngine gates │ └──────────────────────────────┘ │ │ │ ImpedanceController │ │ │ │ Torque actuation │ ┌──────────────────────────────┐ │ │ └──────────┬───────────┘ │ planning_hri │ │ │ │ SPSC digest │ Priority 3 · 20 Hz │ │ │ ▼ │ │ │ │ ┌──────────────────────┐ │ Dreamer re-plan │ │ │ │ idle │ │ HRI consent refresh │ │ │ │ Priority 0 │ └──────────────────────────────┘ │ │ │ │ │ │ │ BatchedMerkleManager│ │ │ │ StreamingMerkleChain│ │ │ │ (1 Flash write/sec) │ │ │ └──────────────────────┘ │ └─────────────────────────────────────────────────────────────────┘ 

Task Priority Table

TaskPriorityFrequencyResponsibilitiesfast_control_loop101 kHzSecureIpc, jitter watchdog, safety assessment, WBC, actuation, telemetry digestsensor_acquisition5500 HzADC DMA refresh, ESKF predict, covariance updateplanning_hri320 HzMotion planning (Dreamer), HRI consent refreshidle0BackgroundBatched Merkle chain Flash offload 

Tiered Merkle Audit Architecture

1 kHz tick digests │ ▼ (SPSC queue — zero-copy) BatchedMerkleManager (RAM, 1024-digest buffer) │ 1 write/sec (sub-root via SHA-256 fold) ▼ StreamingMerkleChain │ ~86,400 writes/day ▼ QspiFlashRing (NOR Flash ring buffer, 1 MiB) 

On a SafetyFault, emergency_flush immediately drains the RAM buffer to Flash, preserving full 1 kHz audit granularity up to the fault instant.

Module Reference

src/main.rs

RTIC application entry point. Defines all tasks, static DMA buffers (.dma_buf section), shared/local resource structs, and the terminal fault handler (execute_hard_safety_fault). Enables Flush-to-Zero FPU mode via NkHal::new().

src/macros.rs

Certification traceability macros:

safety_req!(id) — emits a requirement ID string into .safety_reqs ELF section for offline requirements matrix extraction.

cert_note!(msg) — emits a defmt::trace certification annotation.

src/ipc.rs (v12 new)

SecureIpc — DMA-backed, non-blocking IPC between cores:

Double-buffered transmit/receive using static .dma_buf SRAM regions.

Hardware CRC (STM32H7 CRC unit) validates every packet.

Faults to SafetyFault::IpcCrcMismatch on corruption.

enforce_amp_redundancy() cross-validates main/aux sensor readings.

src/foundation/schemas.rs

Central type definitions:

Physical constants (DOF = 26, CARE_PRESSURE_LIMIT_KPA, MAX_HARDWARE_TORQUE_NM, IPC_MAX_LATENCY_US = 800 µs, MERKLE_BATCH_SIZE = 1024)

SafetyFault enum with 12 variants including WatchdogJitterFault(u32) (v12 new)

Sensor structs: PressureMap, VitalsSnapshot, RedundantSensorState, SubjectEvidence, ConsentRecord

Mission enums: MissionMode, SafetyDecision, MissionContext

Float sanitization: sanitize_sensor_f32, sanitize_torque_array

PressureToDofMapping — maps 8 tactile zones to DOF index slices

src/foundation/mission_policy.rs

MissionPolicy trait for compile-time and runtime policy injection:

HealthcarePolicy — most restrictive (72 Nm max torque, 15 kPa pressure, PUF audit required)

RehabPolicy — relaxed torque (80%) and pressure (120%) limits, no PUF audit

src/foundation/storage.rs (v12 new)

BatchedMerkleManager:

Accumulates 1,024 SHA-256 tick digests in a static RAM array (zero heap allocation).

On full buffer: computes sub-root via iterative SHA-256 folding; writes one 32-byte block to Flash.

emergency_flush() drains the entire RAM buffer to Flash individually for post-mortem reconstruction.

QspiFlashRing:

Circular NOR Flash ring buffer (1 MiB, 32-byte blocks).

Implements NvmStorage trait; performs sector erase before each new sector boundary.

src/foundation/crypto.rs

StreamingMerkleChain:

Maintains a rolling Merkle root over the sequence of sub-roots from BatchedMerkleManager.

Each append() call computes SHA256(current_root || sub_root) and writes to Flash.

At 1 Hz: ~86,400 writes/day — well within NOR Flash endurance limits.

src/control/hal_nk.rs

Full STM32H7 HAL abstraction:

ADC (DMA-backed), SPI, CAN-FD, GPIO

read_pressure_map(), read_vitals(), write_torque_commands()

open_emergency_contactor() — immediate power-off safety line

read_cycle_counter() — DWT cycle counter for jitter measurement

FPU Flush-to-Zero configuration for deterministic float behaviour

src/control/wbc.rs

ImpedanceController — Whole-Body Control with impedance modulation:

apply_tactile_safety_local() — enforces policy-defined pressure and torque limits before any actuation command reaches the motor bus.

Kani-verified: verify_torque_ceiling_not_exceeded and verify_nan_input_returns_error proofs pass.

src/control/physics.rs

Jacobian contact model and pressure prediction for contact-aware motion planning.

src/estimation/eskf.rs

Error-State Kalman Filter (ESKF):

predict(accel, dt) — propagates state at 500 Hz.

uncertainty() — returns scalar covariance norm; gates safety engine if > ESKF_UNCERTAINTY_LIMIT.

src/cognitive/safety_engine.rs (v12 new)

EthicalSafetyEngine — full safety gate sequence executed every tick:

Consent validation

Sensor redundancy cross-check

Pressure limit enforcement

ESKF covariance bound check

Voltage brownout detection

Jitter watchdog evaluation

generate_iwdg_key(now_cycles) — gates IWDG feed on both correct control-flow key and DWT jitter window (±10% of nominal). Returns Err(WatchdogJitterFault) on violation, starving the watchdog.

src/cognitive/dreamer.rs

Motion planning and intent inference. Provides update_plan() at 20 Hz; outputs waypoints consumed by the planning task.

src/hri.rs

Human-Robot Interaction channel:

refresh_consent() — polls the UI consent channel; halts planning on withdrawal.

Session token management.

src/telemetry.rs (v12 new)

build_deterministic_digest(ctx):

Converts f32 sensor values to i32 fixed-point (scaled integers) before SHA-256 hashing.

Sets FPU FPSCR Flush-to-Zero flag.

Produces bit-identical digests across resets, compiler versions, and FPU ABI variants.

Hardware Requirements

ComponentSpecificationMCUSTM32H743 (or compatible STM32H7xx variant)ArchitectureARM Cortex-M7, FPU VFPv4-D16SPFlashQSPI NOR Flash, ≥ 1 MiB dedicated to Merkle ring (MRAM strongly recommended for head node)RAMDMA-accessible SRAM region for IPC buffers (.dma_buf section)PeripheralsCRC hardware unit (required for SecureIpc), DMA controller, IWDG, DWT cycle counterCommunicationCAN-FD (26-axis motor bus), SPI (IPC), ADC with DMAActuatorsUp to 26 DOF servo/motor drivers with emergency contactorSensorsRedundant pressure sensors (8 zones × 2 banks), IMU (I2C LSM6DSO or equivalent), vitals monitor 

Software Requirements

Toolchain

RequirementVersionRust edition2021RTIC2.1 (thumbv7-backend)stm32h7xx-hal0.15 (features: stm32h743, rt, quadspi, crc)cortex-m0.7 (critical-section-single-core)heapless0.8sha20.10 (default-features = false)defmt0.3postcard + serde1.0 (no_std, derive)embedded-hal1.0embedded-storage0.3 

Build Profile (release)

[profile.release] opt-level = "s" # Size-optimised lto = true # Link-time optimisation codegen-units = 1 # Single codegen unit for maximum optimisation panic = "abort" # No unwinding in safety-critical code [profile.release.build-override] rustflags = ["-C", "target-feature=+vfp4d16sp"] # Lock FPU ABI for determinism 

Optional Features

FeaturePurposekaniEnable Kani model-checking x Prusti formal verification contracts 

Safety Certifications & Standards Compliance

Standards

StandardScopeIEC 61508Functional safety of E/E/PE safety-related systemsISO 13482Safety requirements for personal care robots 

IEC 61508 Clause Mapping

ClauseRequirementHARP v12 Implementation§7.4.2Deterministic Signal ProcessingFixed-point build_deterministic_digest; FPU ABI locked via +vfp4d16sp; FTZ FPSCR flag§7.4.7Fault Detection in CommunicationSecureIpc with DMA + hardware CRC; IpcCrcMismatch fault on corruption§7.4.3Audit Trail IntegrityBatchedMerkleManager preserves 1 kHz granularity; emergency_flush on fault 

Certification Infrastructure

Requirements traceability: safety_req!() macros embed requirement IDs in .safety_reqs ELF section, extractable by offline audit tools to produce a full requirements matrix from the binary.

Certification notes: cert_note!() macros emit defmt::trace annotations for boot and operational milestones.

Formal proofs: Kani harnesses provide machine-checked guarantees on torque ceiling and NaN handling (see Formal Verification section).

Tamper-evident audit log: SHA-256 Merkle chain over all tick digests, persisted to Flash with emergency_flush ensuring zero gaps on fault.

Requirements Traceability Matrix

Req IDModuleDescriptionSR-SAFETY-COREcognitive/safety_engine.rsFull safety gate sequence executed each tickSR-CTRL-01cognitive/safety_engine.rsConsent validated before any motionSR-CTRL-02control/wbc.rsTactile pressure limits enforced before actuationSR-RED-01ipc.rsDual-core AMP cross-validation of sensor readingsSR-MONITOR-02cognitive/safety_engine.rsTemporal watchdog resets on each successful tickSR-CRYPTO-01foundation/crypto.rsEvery tick appended to tamper-evident Merkle chainSR-HAL-01control/hal_nk.rsAll ADC reads sanitized before useSR-IWDG-01main.rs + safety_engine.rsIWDG fed only on correct key AND jitter check passSR-IPC-01ipc.rs(v12) Non-blocking DMA exchange with hardware CRC guardSR-DET-01telemetry.rs(v12) Fixed-point scaling for bit-identical Merkle digestsSR-FLASH-01foundation/storage.rs(v12) Batched Flash writes ≤ 1/s to preserve enduranceSR-JITTER-01cognitive/safety_engine.rs(v12) Jitter window enforced; IWDG starved on violation 

Formal Verification

Kani Proofs

HarnessStatusProperty Verifiedverify_torque_ceiling_not_exceededPASS∀ finite inputs: |torque| ≤ max_torque_nmverify_nan_input_returns_errorPASSNaN command → Result::Err(SensorCorruptedNaN) 

Run proofs with:

cargo kani --features kani --harness verify_torque_ceiling_not_exceeded cargo kani --features kani --harness verify_nan_input_returns_error 

Upcoming (v13)

Prusti proofs on sanitize_sensor_f32 and enforce_amp_redundancy for Rust-level pre/post-condition verification.

What HARP v12 Solves

1. Log Death (Flash Endurance Exhaustion)

Problem: In v11, StreamingMerkleChain wrote one 32-byte SHA-256 block to QSPI NOR Flash on every 1 kHz control tick. A standard NOR Flash rated at 100,000 erase cycles would be completely exhausted in approximately 100 seconds of operation — making continuous audit logging physically impossible.

Solution: BatchedMerkleManager accumulates 1,024 tick digests (1 second of data) in a static RAM buffer, folds them into a single sub-root via iterative SHA-256, and writes one 32-byte block to Flash per second. Flash write rate drops from 1,000/s to 1/s — a 1,000× reduction — while full 1 kHz audit granularity is preserved in RAM and flushed in its entirety on any fault.

2. Non-Deterministic Telemetry Digests

Problem: v11 computed Merkle digests by hashing raw f32 sensor values. The IEEE 754 bit pattern of a float depends on the compiler, optimisation flags, FPU ABI, and denormal handling. Identical physical measurements could produce different digest chains across firmware builds or after a reset, making audit log verification unreliable.

Solution: build_deterministic_digest converts all f32 values to scaled i32 fixed-point integers before hashing. The FPU FPSCR Flush-to-Zero bit is set at boot. The release profile locks the FPU ABI to VFPv4-D16SP via rustflags. The result is bit-identical SHA-256 digests for identical sensor values, regardless of compiler version or reset history.

3. Timing-Blind Watchdog

Problem: v11's IWDG was fed solely on a correct control-flow key, verifying execution sequence but not execution timing. A task that ran at the right points but took arbitrarily long (due to cache miss, DMA stall, or priority inversion) would still feed the watchdog, masking real-time deadline violations.

Solution: generate_iwdg_key() gates the IWDG feed on both the correct control-flow key and a DWT cycle-counter jitter check (±10% of the nominal 1 ms period). Any tick that is too slow or too fast raises SafetyFault::WatchdogJitterFault and deliberately starves the IWDG, triggering a hardware reset.

4. Blocking IPC

Problem: v11 used SpiStub, a blocking, simulated SPI transfer that consumed CPU time inside the highest-priority 1 kHz task, inflating worst-case execution time and violating the tight 1 ms budget.

Solution: SecureIpc performs DMA-backed, non-blocking double-buffered transfers using statically allocated .dma_buf SRAM regions. A hardware CRC unit validates every packet. The control loop initiates a transfer and returns immediately; the result is read at the start of the next tick.

Key Novelties in v12

Tiered Merkle Tree with Emergency Flush

The two-level architecture — BatchedMerkleManager (RAM) feeding StreamingMerkleChain (Flash) — is novel in embedded safety firmware. It achieves simultaneous goals that are normally in tension: (a) full-resolution 1 kHz audit granularity for forensic reconstruction, (b) Flash write rates low enough to never exhaust device endurance, and (c) zero audit-trail gaps on sudden faults via the emergency_flush path.

Fixed-Point Deterministic Hashing

Combining fixed-point integer representation, locked FPU ABI (+vfp4d16sp), and FPSCR Flush-to-Zero to achieve bit-identical cryptographic digests across firmware versions and hardware resets is a non-obvious technique. It closes a class of verification failures that would otherwise only manifest during post-hoc log audits.

Latency-Aware IWDG

Coupling a DWT hardware cycle counter to the IWDG feed decision — rather than using only a software key — elevates the watchdog from a control-flow checker to a genuine real-time deadline enforcer, without requiring an external timing supervisor.

Hardware-Verified Non-Blocking IPC

Using the STM32H7's onboard CRC peripheral (rather than software CRC) to validate inter-core packets while DMA handles the transfer yields a path where the highest-priority task pays near-zero CPU cost for both transport and integrity verification.

Zero-Heap, no_std Safety Stack

The entire firmware — including the Merkle batcher, ESKF, WBC, and safety engine — operates with zero heap allocation. All buffers are statically sized using heapless and compile-time constants, eliminating allocator failure as a fault mode.

Potential Applications

Medical & Assistive Robotics

Exoskeletons for mobility-impaired individuals — the consent gate, torque limits, and pressure map enforcement make HARP directly applicable to upper- and lower-limb exoskeleton control.

Rehabilitation robots — RehabPolicy provides a ready-made relaxed-limit profile for therapeutic motion under clinical supervision.

Surgical assist devices — the tamper-evident Merkle audit trail satisfies medical device traceability requirements; the jitter watchdog ensures sub-millisecond timing guarantees required for haptic feedback.

Industrial Human-Robot Collaboration

Collaborative robot arms operating in close proximity to workers — the redundant sensor fusion, torque saturation, and emergency contactor logic map directly to ISO/TS 15066 power-and-force limiting requirements.

Wearable industrial assist suits — low-DOF subsets of the 26-DOF model can be used for load-bearing or ergonomic assist devices.

Research & Development Platforms

Formal verification research — the Kani and Prusti integration points make HARP a reference platform for studying machine-verified real-time control firmware.

Safety-critical embedded Rust — the codebase demonstrates production patterns for no_std, RTIC-based, IEC 61508-targeting firmware and can serve as an educational or template codebase.

Haptics research — the Jacobian contact model and 8-zone pressure map provide infrastructure for studying contact-rich manipulation.

Emergency & First-Responder Equipment

The MissionMode::Emergency variant and emergency contactor logic support deployment in first-responder exoskeletons or patient-handling devices where rapid abort and power-off are mandatory.

v11 → v12 Improvement Summary

Issuev11 Statusv12 SolutionIPCSpiStub (blocking simulation)SecureIpc — DMA + hardware CRC; non-blocking double-buffered exchangeLog Death1 kHz NOR Flash writes (exhausted in ~100 s)BatchedMerkleManager — 1 write/sec to Flash; MRAM recommended for head nodeFP DeterminismRaw f32 hashing (compiler-dependent bit patterns)build_deterministic_digest — fixed-point i32 scaling + FTZ FPSCR flag + locked FPU ABIWatchdogKey-based IWDG feed only (timing-blind)JitterWatchdog — ±10% DWT cycle window; IWDG starved on jitter violation 

v12 → v13 Roadmap

Prusti proofs on sanitize_sensor_f32 and enforce_amp_redundancy for Rust-level formal verification of input sanitisation contracts.

defmt-test integration for on-device unit tests covering pressure map parsing, ESKF bounds, and CRC correctness.

Secure boot: verify firmware Merkle root against OTP-fused expected root at init, closing the firmware substitution attack surface.

Power management: VBAT brownout graceful shutdown — contactor open + emergency_flush + NVM sync — replacing the current hard-halt.

EtherCAT support for the 26-axis motor bus as an alternative to CAN-FD, enabling sub-100 µs cycle times on large DOF configurations.

MRAM integration: replace QspiFlashRing head-of-chain writes with an MRAM-backed NvmStorage impl (10¹⁴ write cycles vs 10⁵ for NOR Flash), eliminating the last Flash endurance concern.

Jacobian calibration pipeline: offline robot-model tool generating static physics.rs table for each hardware variant.

Build & Verification

Build

# Release build (size-optimised, LTO, locked FPU ABI) cargo build --release --target thumbv7em-none-eabihf # Debug build cargo build --target thumbv7em-none-eabihf 

Flash

# Using probe-rs probe-rs run --chip STM32H743ZITx target/thumbv7em-none-eabihf/release/harp 

Run Kani Proofs

cargo kani --features kani --harness verify_torque_ceiling_not_exceeded cargo kani --features kani --harness verify_nan_input_returns_error 

Extract Requirements Matrix

The .safety_reqs ELF section contains all safety_req!() markers as HARP_REQ:<ID>:<file>:<line> strings. Extract with:

arm-none-eabi-objdump -s -j .safety_reqs target/thumbv7em-none-eabihf/release/harp 

HARP v12 · By James Squire· IEC 61508 / ISO 13482 Compliant

