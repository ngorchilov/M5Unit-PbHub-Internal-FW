# PBHUB Firmware Remediation Implementation Plan

> Status: working project document. This plan and its companion engineering
> reference are not proposed for upstream publication in their current form.

This plan turns the findings in the
[firmware protocol and engineering reference](pbhub-firmware-protocol.md) into a
series of independently reviewable upstream changes. Every phase below is the
boundary of exactly one commit and one upstream pull request.

## Documentation and upstream strategy

The working documents explain how this project was derived and coordinated, but
their audit narrative, temporary phase identifiers and unfinished validation
backlog do not need to become permanent upstream documentation.

Use the following publication boundary:

- Keep these two working documents on a private or local planning branch, or in
  a separate companion repository. Do not include them in implementation PR
  branches merely because they motivated the change.
- Do not open a documentation-baseline PR. Start upstream work with PR-01.
- Include only durable documentation required by the specific implementation
  in each PR. Examples are reproducible build instructions in PR-01, memory-map
  constraints in PR-02, and stable protocol definitions in PR-11 and PR-13.
- Put transient motivation, investigation notes, measurements and the checklist
  for a phase in its upstream PR description rather than in the source tree.
- At project completion, either archive the working documents outside upstream
  or deliberately distill their still-useful parts into stable user/developer
  documentation. Do not merge the working documents unchanged by default.

For this checkout, the safest Git workflow is to retain a planning branch that
contains the working documents. Create every implementation branch from clean
upstream `main` (or from a merged prerequisite), then copy only the relevant
code, tests and durable documentation into its single commit. This prevents the
planning documents from leaking into upstream PRs.

## Rules for every implementation phase

- One phase equals one commit and one upstream PR.
- Code, focused tests and durable documentation for that scope travel in the
  same commit.
- Do not include unrelated cleanup, phase numbers or links to the private
  planning documents in production sources.
- Preserve every previously valid V2 command unless the PR explicitly defines
  and justifies a compatibility change.
- Record source tests and hardware measurements in the PR description.
- Do not merge a phase until its acceptance boundary is satisfied.

## Release gates and dependencies

PR-01 and a successful full-flash/option-byte backup and factory-image restore
over SWD are gates for every firmware-changing PR.

PR-10 is a prerequisite for PR-11. PR-12 must land before any legacy IAP path is
exposed to normal users. PR-13 is a separate protocol-design effort and must not
be bundled with legacy IAP hardening.

## Finding ownership

| Finding | Owning phase |
|---|---:|
| Missing reproducible CRC, version-byte, image merge and layout inspection | PR-01, PR-02 |
| Linker regions cross bootloader, application, CRC and settings boundaries | PR-02 |
| RGB single/fill validation permits out-of-bounds access and premature mode changes | PR-03 |
| Invalid register alias, inconsistent lengths and mutation before validation | PR-04 |
| Mode replacement, stale edge state and interrupt/main-loop shared state | PR-05 |
| Nonlinear brightness, incomplete buffer clearing and undefined RGB packing | PR-06 |
| RX wrap, stale responses, semantic ambiguity, empty error handling and repeated recovery | PR-07 |
| Reserved I2C addresses, unnecessary erases and non-atomic live-address change | PR-08 |
| Unbounded clock/ADC waits, ignored initialization failure and stale ADC response | PR-09 |
| Main-loop-polled fixed-frequency PWM and servo timing | PR-10 |
| Missing configurable PWM frequency and capability discovery | PR-11 |
| Unsafe legacy IAP frame, address, length and page programming behavior | PR-12 |
| Missing acknowledged, recoverable and versioned IAP protocol | PR-13 |

## Implementation phases

| Phase | Single-commit/upstream-PR scope | Acceptance boundary |
|---:|---|---|
| PR-01 | Add reproducible image inspection and assembly tooling | Verify all Intel HEX checksums; reproduce the 16 KiB region map; pad the CRC range with `0xFF`; calculate `0xCE41DBAC` for the factory application; emit and re-parse a merged image without changing firmware behavior |
| PR-02 | Enforce flash/RAM ownership and explicit image metadata | Bootloader cannot link beyond `0x08000FFE`; bootloader version occupies `0x08000FFF`; application code cannot reach the CRC slot at `0x08003BFC` or settings at `0x08003C00`; the `0xC0` SRAM vector reservation remains enforced; PR-01 checks pass |
| PR-03 | Fix RGB single/fill validation and memory safety | Reject invalid index/range before caching, changing mode or touching GPIO; prove boundaries 0, 73 and 74 plus overflow cases with guard-backed tests; valid RGB output remains byte-compatible |
| PR-04 | Make register, length and value validation precede mutation | Remove the `0x00` channel-0 alias; enforce exact lengths; invalid servo and other malformed commands leave mode, caches and pins unchanged; valid command responses remain compatible |
| PR-05 | Centralize deterministic per-signal mode transitions | One internal transition path owns GPIO setup, disable flags, pulse values and edge-state reset; concurrency between ISR and timing engine is explicit; transition-matrix hardware tests pass |
| PR-06 | Correct RGB brightness and portability defects | Use defined unsigned packing; clear all six buffers; define linear rounding/saturation and whether a brightness write retransmits; validate representative values 0, 1, 127, 128, 254 and 255 on hardware |
| PR-07 | Harden application I2C framing, responses, errors and recovery | No RX wrap; every selected read has a deterministic response; short/over-reads have defined cursor behavior; semantic failure is observable; error flags are cleared; stuck-write recovery runs once per incident; all valid V2 host-library calls still pass |
| PR-08 | Make I2C-address persistence safe and wear-aware | Reject reserved/out-of-range addresses, skip unchanged writes, verify the record before switching live address and retain/recover the prior address on failure; power-cycle tests pass |
| PR-09 | Make startup and ADC failure bounded and observable | Clock/ADC waits have defined reset or error behavior; each ADC sample has a bounded wait; failure never reuses a stale valid sample; successful ADC filtering and wire format remain unchanged |
| PR-10 | Move fixed-frequency PWM and servo generation to a timer-driven engine | Preserve the V2 default PWM duty contract and 50 Hz servo contract while eliminating main-loop edge polling; quantify jitter under ADC, RGB and I2C stress before and after |
| PR-11 | Add backward-compatible PWM frequency control | Settle global-versus-per-signal ownership, register allocation, range, resolution, phase, zero/full-scale duty, readback, reset default and version/capability discovery; default behavior matches V2; close or update upstream issue #1 with measured results |
| PR-12 | Bound and verify the existing IAP page-write path | Preserve legacy opcodes while requiring a complete frame, exact 1024-byte count, aligned pages 4 through 14, unsigned packing and post-program verification; page 15 and out-of-flash writes are impossible |
| PR-13 | Add a versioned acknowledged IAP protocol | New opcodes provide bounded frames, status, CRC, read-back verification, explicit finalize/abort and documented recovery; old `0x06`/`0x77` behavior remains isolated; end-to-end interrupted-update tests pass on a spare unit |

Keep [upstream issue #1](https://github.com/m5stack/M5Unit-PbHub-Internal-FW/issues/1)
linked through PR-11 until configurable PWM is available in a released firmware
image.

## Hardware validation backlog

These observations must be measured before the relevant source-derived behavior
is treated as device-verified:

1. Confirm full-flash and option-byte backup, mass erase and factory-image
   restore over the retail SWD pads before testing modified firmware.
2. Reproduce both ARMCC 5.06u6 builds and record binary/map differences from the
   factory image before treating a new toolchain as equivalent.
3. Measure PWM frequency, high time and jitter at duties 0, 1, 127, 254 and 255.
4. Test PWM re-entry after digital, ADC, servo and RGB mode changes.
5. Measure servo pulse width and frame rate at 500, 1500 and 2500 microseconds.
6. Test PWM and servo while polling ADC and updating 74 LEDs.
7. Verify RGB color order, brightness levels and both LED timing modes on
   representative supported LEDs.
8. Exercise RGB invalid-index and boundary cases with RAM guards in a test build.
9. Test digital inputs while floating, driven low and driven high.
10. Measure ADC accuracy, noise and timeout behavior at safe known voltages on
    all six channels.
11. Exercise I2C at 100 kHz and 400 kHz, including repeated starts, exact and
    short reads, malformed frames, overflow, reconnect and injected bus errors.
12. Test address changes across unchanged writes, invalid addresses, reset and
    interrupted flash programming on a recoverable device.
13. Test power sequencing when the host and PBHUB do not start simultaneously.
14. Test legacy I2C IAP entry, page programming, interrupted-update recovery and
    application return through the intended host controller stack.
