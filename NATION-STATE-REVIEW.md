# OPERATION IRON VAULT - Nation-State Accuracy Review

**An adversarial, evidence-based audit of the entire project where every claim is
verified by a re-runnable command or explicitly labelled as a limitation.**

***
**LEGAL DISCLAIMER:**
The information, tools, and code provided in this repository and course are strictly for educational, research, and defensive purposes only. 

You are explicitly prohibited from using any materials contained herein to access, test, modify, or exploit any device, network, or system that you do not own 100% or for which you do not have explicit, documented, and legally binding authorization to interact with.

By using this repository and course, you acknowledge and agree that:

1. Any illegal, unauthorized, or malicious use of this information is solely your responsibility.
2. The author(s) and contributor(s) of this repository and course shall not be held liable for any damages, legal repercussions, criminal charges, or unauthorized actions resulting from the use, misuse, or abuse of the contents herein.
3. You will comply with all applicable local, state, national, and international laws regarding cybersecurity and computer fraud.

**IF YOU DO NOT AGREE WITH THESE TERMS, DO NOT USE THIS REPOSITORY AND COURSE.**

***

## 1. Scope and Method

This review treats the project as hostile-to-itself. Every module, constant, test
vector, document, and artifact is independently checked. The method is:

1. **Re-run every gate** (`audit_c_standard`, `audit_python_standard`,
   `run_tests`, `check_coverage`) and record the exact output and exit codes.
2. **Re-verify every constant** against the generated artifact header and the
   JSON source of truth, by regenerating the header and diffing it.
3. **Re-verify every cryptographic claim** against a published standard vector,
   and separate vectors that run in the native C suite from vectors that run only
   in the Python suite.
4. **Audit the locker adversarially**, with a dedicated Availability and Ransom
   Logic section: what it does, how it holds the vent, how it masks the state,
   how it persists across a reflash, how it is detected and removed, how it is
   bounded, and what it does not prove.
5. **Re-verify every artifact** by SHA-256 and by the in-repo build guardrail,
   because this project ships no CTF firmware artifact.
6. **Read the documents adversarially** for overclaims, stale numbers, and
   omissions, then correct them in this review.

## 2. Gate Results (all re-run for this review)

| gate | command | observed result |
|---|---|---|
| C standard | `python3 scripts/audit_c_standard.py` | exit 0, no output, **0 violations** |
| Python standard | `python3 scripts/audit_python_standard.py` | exit 0, no output, **0 violations** |
| Native tests | `python3 scripts/run_tests.py` | **488 checks, 0 failures**, 145 test cases |
| Coverage | `python3 scripts/check_coverage.py` | exit 0, **100.00% line coverage**, 2082 owned lines |
| Python suites | `python3 -m unittest test.test_field_crypto test.test_vent_node` | **17 tests, OK** |
| Header guardrail | `python3 scripts/gen_packet.py --from-json scripts/packet_artifact.json --header-out /tmp/iv_psa.h --check-header-path include/packet_artifact.h` | exit 0, `Verified header matches` |

The coverage gate passes on line coverage. It does not require 100% branch or
region coverage, and the raw report is not 100% there: regions 99.25% and
branches 93.75%. That gap is real and is stated in the module table below.

## 3. Module-by-Module Audit

Owned lines are the instrumented statement lines reported by `llvm-cov report`
through `check_coverage.py`. Raw `wc -l` over `src/*.c` includes comments and
blank lines; `main.c` is excluded from coverage by design. Every owned module is
at 100.00% line coverage.

| module | role | owned lines | line coverage | verification performed | honest limitation |
|---|---|---|---|---|---|
| `crc.c` | CRC-16/CCITT-FALSE diagnostic | 20 | 100.00% | `test_crc16_ccitt` check value `0x29B1` | not on the wire; a checksum is not authentication |
| `sensor.c` | DHT11 rack-temperature-band classifier | 151 | 100.00% | waveform, all timeout shapes, CRC error, rack ok/reject/invalid, negative temperature | mock GPIO replays a recorded waveform, not real silicon |
| `display.c` | HD44780 over PCF8574 | 73 | 100.00% | `test_display_format_lines`, `test_display_render_lines` via recorded I2C | mock I2C, not real HD44780 bus timing |
| `radio.c` | RYLR998 provisioning, `AT+SEND`, `+RCV` parser | 187 | 100.00% | build/parse/reject/pump, oversize guards, spoofed sender attribution | mock UART; the RF band is not simulated |
| `status_led.c` | red/yellow/green HALL HOT/LOCKED/COOLING OK tower light | 14 | 100.00% | `test_status_led_show` | none |
| `button.c` | manual purge/debounce input | 35 | 100.00% | pressed/consume/debounce/reset | active-low input is exercised only through mocks |
| `servo.c` | 50 Hz damper PWM | 26 | 100.00% | `test_servo_map`, `test_servo_init`, `test_servo_actuate` | mock PWM; no real servo or inrush load |
| `ir_remote.c` | VS1838B NEC maintenance decode | 116 | 100.00% | decode valid/reject/bad leader/mark/ambiguous/address/command, `test_ir_poll_*` | optical path is unauthenticated; no anti-replay |
| `damper.c` | vent damper state machine and fail-open policy | 41 | 100.00% | `test_damper_init`, open/close travel, `test_damper_fail_safe`, `test_damper_reject_unauthorized` | bounded travel only; no real vane or load |
| `control.c` | sealed vent command path | 96 | 100.00% | `test_control_handle_success`, bad command, bad zone, bad tag, replay, short body, key guards | shared lab key; guarded set is three commands |
| `vault_auth.c` | anti-replay window and state tag | 74 | 100.00% | `test_vault_auth_state_tag`, apply window/advance/bad tag, null, key, and tag guards | deterministic nonce from sequence; single key |
| `chacha20.c` | ChaCha20 and HChaCha20 | 99 | 100.00% | RFC 8439 block and stream vectors, HChaCha20 draft vector | none |
| `poly1305.c` | Poly1305 one-time authenticator | 169 | 100.00% | RFC 8439 tag vector, aligned path | none |
| `crypto_aead.c` | XChaCha20-Poly1305 seal/open | 38 | 100.00% | round-trip, tamper tag/ct/ad, constant-time `tag_equal` | built from the in-repo primitives, not an audited library |
| `blake2b.c` | BLAKE2b and Argon2 H' | 161 | 100.00% | `test_blake2b_abc`, multiblock, H' 32 and 256 vectors | none |
| `argon2.c` | Argon2id core (BLAMKA, hybrid addressing) | 336 | 100.00% | `test_argon2_lanes`, `test_argon2_type_i`, `test_argon2_clamp` branch coverage | the RFC 9106 KAT runs in Python, not in this C suite |
| `crypto_kdf.c` | Argon2id field key derivation | 28 | 100.00% | reject, empty password, determinism, salt sensitivity | classroom profile `t=3 p=1 m=64`; committed passphrase and salt |
| `envelope.c` | hex nonce/ciphertext/tag codec | 91 | 100.00% | nonce, round-trip, seal/open rejects, uppercase, known vector | none |
| `monitor.c` | vent controller state machine | 253 | 100.00% | init, idle, render (including lock), temperature, maintenance remote, remote open/close/purge/replay/bad tag/bad command/bad zone, purge no-bypass, link loss, link unseen/within, guards, locker frame delivery and mask | mocks are not the real silicon |
| `implant.c` | SANDBOX_ONLY FROSTLINE ransom locker | 74 | 100.00% | first run, re-lock on boot, lock marker, debug attached, forced close, tick armed/unarmed/debug, release accept/reject/debug, neutralize, clean tick | benign educational locker; build-guarded and breadboard-bound |
| `main.c` | entry point | n/a | excluded | build only | excluded from coverage by design |

**Total owned lines at 100.00% line coverage: 2082.**

Branch coverage below 100% in the same report: `monitor.c` 86.05%, `display.c`
85.71%, `radio.c` 89.87%, `control.c` 90.48%, `vault_auth.c` 92.31%,
`envelope.c` 92.86%, `sensor.c` 94.74%, `ir_remote.c` 96.00%, `argon2.c` 97.56%.
`implant.c` branches are 100.00%.

## 4. Cryptographic Claim Verification

The native suite asserts the following published vectors. Each name below appears
as a passing case in the `run_tests.py` output for this review.

| claim | standard | vector | observed |
|---|---|---|---|
| ChaCha20 block function | RFC 8439 section 2.3.2 | key 00..1f, nonce 000000090000004a00000000 | `test_chacha20_block` PASS |
| ChaCha20 stream cipher | RFC 8439 section 2.4.2 | "Ladies and Gentlemen..." 114-byte ciphertext | `test_chacha20_stream` PASS |
| HChaCha20 subkey | XChaCha20 draft (irtf-cfrg-xchacha) | published subkey vector | `test_hchacha20` PASS |
| Poly1305 tag | RFC 8439 section 2.5.2 | "Cryptographic Forum Research Group" tag `a8061dc1305136c6c22b8baf0c0127a9` | `test_poly1305`, `test_poly1305_aligned` PASS |
| BLAKE2b-512 | BLAKE2 reference | digest of "abc", multiblock, long-input | `test_blake2b_abc`, `test_blake2b_multiblock` PASS |
| Argon2 variable-length hash H' | RFC 9106 section 3.3 | H' of {1,2,3,4} at 32 and 256 bytes | `test_blake2b_long_short`, `test_blake2b_long` PASS |
| Argon2id known-answer | RFC 9106 section 5.3 | `0d640df58d78766c08c037a34a8b53c9d01ef0452d75b65eb52520e96b01e659` | `test.test_field_crypto.TestFieldCrypto.test_rfc9106_argon2id_vector` PASS (Python suite) |
| Envelope layout | project vector | known nonce, node id 7, fixed body | `test_envelope_known_vector` PASS |
| Firmware and Python interop | project vector | shared field key and envelope | `test_field_key_matches_firmware`, `test_envelope_matches_firmware` PASS (Python suite) |

The RFC 9106 Argon2id known-answer test is a Python `unittest` in
`test/test_field_crypto.py`; it is not part of the 488 native checks. Running the
Python suites directly confirms all 17 tests pass, including the KAT and the
firmware-interop vectors.

### 4.1 Sealed vent path, guarded set, and bounded zone band

`src/control.c` opens the envelope under the field key with the vent node id as
associated data, then `control_parse` rejects the body unless the command byte is
one of `VENT_COMMAND_OPEN` (`0x01`), `VENT_COMMAND_CLOSE` (`0x02`), or
`VENT_COMMAND_PURGE` (`0x03`), and the decoded zone lies between `VENT_ZONE_MIN`
(`0`) and `VENT_ZONE_MAX` (`16`). The guarded set is therefore exactly those
three commands, and the accepted zone band is exactly 0 to 16. The behavior is
asserted by `test_control_handle_success`, `test_control_command_set`,
`test_control_authorize`, `test_control_replay`, `test_control_bad_command`,
`test_control_bad_zone`, `test_control_bad_tag`, `test_control_short_body`, and
`test_control_key_guards`. All pass in this review.

Note that `scripts/gateway.py` sends `VENT_COMMAND_OPEN = 1`, which agrees with
the firmware decoding `0x01` as open, and `scripts/spoof.py` forges
`VENT_COMMAND_CLOSE = 2`, which agrees with the firmware decoding `0x02` as
close. There is no tooling/firmware command-constant mismatch in this act.

### 4.2 Anti-replay sequence window

`vault_auth_apply` in `src/vault_auth.c` accepts a command only when
`seq > auth->last_seq`, then verifies the keyed tag against the candidate record,
then advances the floor. The behavior is asserted by
`test_vault_auth_apply_window` (accept once, reject the same sequence, reject an
older sequence), `test_vault_auth_apply_advance` (a newer sequence advances
`last_seq`), `test_vault_auth_bad_tag`, and the end-to-end
`test_monitor_remote_replay`. All pass in this review.

Honest limitation: `last_seq` is plain SRAM and resets to zero on every boot, so
a command captured before a reboot can be replayed after one. The paper's Threat
Model states this; the README does not. A production controller would persist the
floor in non-volatile memory.

### 4.3 Authenticated state tag

`vault_auth_state_tag` seals a nine-byte record (`granted`, `seq[4]`,
`last_seq[4]`) under the field key with a nonce built from the sequence and the
domain byte `0xA7`; `vault_auth_state_ok` recomputes and compares in constant
time (`crypto_aead_tag_equal` is a branchless XOR accumulator).
`test_vault_auth_state_tag` proves a modified record fails, and the monitor
paths prove the end-to-end denial before the damper moves.

Honest limitations: the lab derives the field key and the state-tag key from one
committed secret, so a compromised device can compute tags the gateway accepts;
the tag protects against casual tamper and a debugger that flips `granted`, not a
physical attacker who can read the key out of SRAM and recompute the tag; there
is no per-device key or rotation; and the nonce is deterministic in the sequence,
so two distinct records that ever share a sequence would violate AEAD nonce
uniqueness.

## 5. Availability and Ransom Logic Audit

Act VIII is the availability act, so the locker gets its own dedicated audit.
The review asks six questions: what it does, how it holds the vent, how it masks
the state, how it persists, how it is detected and removed, how it is bounded,
and what it does not prove.

### 5.1 What it does

`src/implant.c` is compiled only under `SANDBOX_ONLY`. The clean firmware build
does not define the guard, so the shipping image has no locker. In the
`SANDBOX_ONLY` build:

- **Lock marker with the real flash API.** `implant_init` reads the marker byte
  at `VENT_IMPLANT_RESERVE_ADDR` (`0x103FF000`). On the first run the marker is
  absent, so `implant_infect` erases the sector and programs `0x4C`
  (`VENT_IMPLANT_LOCK_MARKER`) with the Pico SDK flash API exactly once. This is
  a real sector erase and program: in the non-mock build the `IMPLANT_FLASH_WRITE`
  macro expands to `flash_range_erase(VENT_IMPLANT_RESERVE_OFFSET,
  FLASH_SECTOR_SIZE)` followed by `flash_range_program(VENT_IMPLANT_RESERVE_OFFSET,
  page, FLASH_PAGE_SIZE)` against the final 4 KiB sector of the 4 MiB flash, not a
  simulated memory-mapped store. `implant_marker_set` reports the marker by
  comparing the reserved byte to `0x4C`.
- **Re-lock on boot.** On every later boot the marker is present, so `implant_init`
  sets the active and locked latches and the forced close is live again without
  any firmware change. A reflash of the program region does not touch the
  reserved sector.
- **The forced close.** While `implant_lock_active` is true,
  `monitor_open_target` returns false, so `monitor_apply_state` drives the damper
  closed regardless of the guarded vent state. The locker never builds an
  envelope and never presents a key; it overrides the output after the
  authenticated decision.
- **The maintenance mask.** `monitor_state_text` returns `MAINT` while the lock
  is active, so `monitor_format_lines` renders `ST:MAINT` and `monitor_led_for`
  returns `VENT_LOCKED`, lighting the yellow lamp. The operator is told the vent
  is maintenance locked.
- **Autonomous re-assertion.** `implant_tick` advances a monotonic counter and,
  while the locker is active and no probe is attached, re-asserts the closed
  position every `VENT_IMPLANT_TICK_INTERVAL` (4) ticks.
- **The magic release token.** `implant_release` calls `implant_token_ok`, which
  is a `memcmp` against the exact 18-byte `VENT_IMPLANT_RELEASE_MAGIC`
  (`VAULT-RELEASE-2026`). A wrong token, a null pointer, a wrong length, or an
  attached probe leaves the vent locked. On success it clears the latches and
  writes zero to the reserved sector.
- **Anti-debug.** `implant_debug_attached` reads CoreDebug `DHCSR` at
  `0xE000EDF0` (`VENT_IMPLANT_DHCSR_ADDR`). Bit 0 is `C_DEBUGEN` and bit 1 is
  `C_HALT`. `implant_init`, `implant_lock`, `implant_tick`, and
  `implant_release` all return early when either bit is set, so the forced close,
  the mask, and the release are suppressed while a probe is attached.

Every one of these behaviors is asserted by a native test:
`test_monitor_implant_lock_mask`, `test_monitor_implant_release_restores`,
`test_implant_init_first_run`, `test_implant_reinstall_on_boot`,
`test_implant_marker`, `test_implant_debug_attached`, `test_implant_init_debug`,
`test_implant_lock_debug`, `test_implant_tick_lock`, `test_implant_tick_debug`,
`test_implant_tick_unarmed`, `test_implant_release_accepts`,
`test_implant_release_rejects`, `test_implant_release_debug`, and
`test_implant_neutralize`. The Python adapter additionally asserts the release
with `test_09_implant_release_accepts`. All pass in this review.

### 5.2 How it holds the vent, and why a green lamp is not a working vent

The lock is deliberately local and deliberately quiet. An armed, unobserved node
drives the damper closed and renders `ST:MAINT` with the yellow lamp, even when
the guard state would open the vent. Nothing in the path opens an envelope,
presents a key, or changes a byte that an AEAD tag covers, so an integrity check
over content is blind to it. The state is the `0x4C` marker in the reserved
sector and the runtime latches in SRAM, and the loop is the four-tick
re-assertion. A firmware reflash writes the program region and leaves the marker
alone, so the node comes back locked.

### 5.3 How it masks the state

The mask is the half of the attack that makes it survivable for the attacker. A
forced close that reported `HOT` would be a fault an operator would investigate.
A forced close that reports `MAINT` is a story an operator accepts. The mask is
one function, `monitor_state_text`, and it is compiled under the same
`SANDBOX_ONLY` guard, so the clean build always reports the true guarded state.
The lesson is that availability failures are often social as much as technical:
the device does not only refuse, it explains the refusal.

### 5.4 How it stays underneath the authenticated path

The locker is not a stealth protocol client. It never builds an envelope, never
holds a key, and never calls `control_handle_frame`. Its entire effect is a
boolean read in `monitor_open_target` and a string in `monitor_state_text`. That
is the architectural point: the sealed vent path can be correct, tested, and
replay-resistant, and a local condition that changes the output is unaffected by
every one of those properties. A valid open command can arrive, pass every
check, and the damper will still stay shut.

### 5.5 How it is detected and removed

- **By build comparison.** The clean and `SANDBOX_ONLY` images differ by the
  locker translation unit and its symbols, which is the simplest and strongest
  detection: the payload is absent from the shipping build.
- **By reserved-sector inspection.** The marker at `0x103FF000` is state the
  firmware image does not own, and it is visible with the Debug Probe or
  `picotool`. Because it is written with the real flash API, a read of the sector
  returns the `0x4C` byte on physical silicon.
- **By display mismatch.** The `K:LOCK` lock field and the `ST:MAINT` state next
  to a quiet control link are the locker's fingerprint.
- **By static analysis.** The `0x4C` marker, the `VAULT-RELEASE-2026` token, the
  tick interval, and the `DHCSR` read address are all literal constants in the
  image.
- **By controlled observation.** Because the anti-debug branch is a single early
  return, a student can break after it under GDB and observe the forced close and
  the mask resume, which proves the payload rather than merely suspecting it.
- **By removal.** The documented fix is not one step: break the lock with the
  token, clear the runtime state and the marker (`implant_neutralize`), erase the
  reserved sector, remove the re-lock check and the `SANDBOX_ONLY` build flag,
  and add a fail-open policy so no future boot trusts the marker.

### 5.6 How it is bounded

The locker is bounded by construction and by test. It touches only its own
outputs, its runtime latches, and the one reserved sector. It has no network, no
filesystem, and no host impact. Its only physical effect is on the mock vent
servo and the mock LCD readout. It never reads the field key, never opens the
sealed path, and never writes anywhere except the reserved sector on the same
chip. The `SANDBOX_ONLY` guard is the containment boundary, the reserved sector
holds nothing else, and the native locker tests assert both the behavior and its
limits.

### 5.7 What it does not prove, and the honest limitation

The locker is a **benign educational ransom locker**. It is confined to the
breadboard, guarded by `SANDBOX_ONLY`, and has no network. **The lock affects
only the mock vent servo and the mock LCD readout.** The hall does not in fact
heat and the racks do not throttle; the temperature is synthetic and the vent is
a servo. **It holds no real data.** The lock marker is a real sector erase and
program, but it is a single byte in a reserved sector that holds nothing else.
There is no external address, no internet path, no remote server, and no
command-and-control endpoint. The release token is a literal constant, the lock
is a boolean, and the mask is one string. **No external network exists** anywhere
in this project. It is a demonstration of technique, not tradecraft: it does not
encrypt anything, it does not randomize its token, it does not survive a
deliberate sector erase, and it does not resist physical forensics. Any claim
that this module is operationally representative of real ransomware or a real
availability attack would be an overclaim, and this review records that plainly.

The deeper honest limitation is architectural and does not go away with a cleaner
implementation: a local condition that overrides an authorized output is not a
wire-authentication problem, and no amount of sealing the command path fixes it.
Mitigating it is an authorization-boundary, policy, state-erasure, and
debug-lockdown control, which is why the blue half names those controls rather
than pretending the protocol covers them.

## 6. Artifact Verification

This project ships no CTF firmware artifact (`build/` holds only untracked local
test binaries). The companion CTF is external:
`https://github.com/mytechnotalent/CTF_datacenter-vent-controller`, which ships
the compromised image with the locker and its verifier. What is verified in this
repository is the source tree and the provisioning artifact.

Source-tree aggregate SHA-256 over all 44 `.c` and `.h` files under `src/` and
`include/`, computed as `find src include \( -name '*.c' -o -name '*.h' \) |
sort | xargs shasum -a 256 | shasum -a 256`:

```
96488df6530d6b434ff21b649d444b7999eff50a54d9e5c0826866216c06a0f7
```

Key artifacts by SHA-256:

```
paper.pdf                     7f69b81b84576298c6312286f92559ea8567176f031847c2d5f71e37675e102f
paper.typ                     1be2fc97fdbb038a83316f3c9916285632adf282bc226bf626addd3d091d9b74
datacenter-vent-controller.png 3ec1c3ba02e63becfa86acfefdb6aeceac5cb6462033163d6ef388a3b1711182
scripts/packet_artifact.json  44cef78d7a68e51dc6348b61a36a93c644b2dc004b8f5eac04c1d5747afa6273
include/packet_artifact.h     ea89858d24d047df523fd1359736ce46de04755ce6ccfc35fb3c6a0238fbd0e4
include/field_secrets.h       63cffbbeb9e4740c4031865d6bcf5bb8302ede090579eb4fbf0ef41764135d07
```

The build guardrail `check_packet_artifact_header` regenerates
`include/packet_artifact.h` from `scripts/packet_artifact.json` and fails if the
committed header is stale. Re-run for this review:

```
$ python3 scripts/gen_packet.py --from-json scripts/packet_artifact.json \
      --header-out /tmp/iv_psa.h --check-header-path include/packet_artifact.h
Wrote generated firmware header: /tmp/iv_psa.h
Verified header matches: .../include/packet_artifact.h
exit=0
```

Constants re-read from `include/packet_artifact.h` and matched to the README and
the pin map: `PACKET_NODE_ADDRESS` 7, `PACKET_GATEWAY_ADDRESS` 0x0001,
`PACKET_FRAME_SIZE` 48, `PACKET_LINK_WAIT_MS` 5000,
`PACKET_VENT_OPEN_PULSE_US` 1500, `PACKET_VENT_CLOSE_PULSE_US` 500,
`PACKET_DHT_TIMEOUT_US` 240, `PACKET_LCD_I2C_ADDRESS` 0x27,
`PACKET_MAX_RCV_LEN` 256, plus the shared pin map.

The banner is generated by `scripts/gen_banner.py` and is 1500 x 1500 pixels,
matching the other acts.

## 7. Adversarial Document Review

| document claim | audit verdict |
|---|---|
| README does not imply the device is unhackable | **accurate**; it states the vent path is sealed, that the locker overrides the output underneath it, and that removal is not a single patch |
| README states the locker is benign, guarded, and confined | **accurate**; it appears in the narrative, the locker section, Lab 3, PARTS.md, and this review |
| README states the lock affects only the mock vent and LCD and holds no real data | **accurate**; it is stated in the narrative, the FROSTLINE Ransom Locker section, Lab 3, PARTS.md, and this review |
| README gives the marker, token, interval, mask, and anti-debug details | **accurate**; the `0x4C` marker, `VAULT-RELEASE-2026` token and its 18-byte length, the 4-tick interval, `ST:MAINT`, the `DHCSR` bits, and the reserved address all match the firmware |
| README states the clean build does not define `SANDBOX_ONLY` | **accurate**; the CMake option defaults to OFF and the module is guarded |
| README "The suite has **145 cases** and **488 checks**" | **accurate**; the native runner reports exactly 145 cases and 488 checks |
| README claims 100% line coverage of owned modules | **accurate**; the report is 2082 / 2082 lines |
| README claims the lock marker uses the real flash API | **accurate**; the non-mock `implant_flash_write` calls `flash_range_erase` and `flash_range_program` |
| README does not mention per-device key rotation | **omission**; the paper's Threat Model is the only place that states the single-key limitation |
| README anti-replay section does not mention the reboot reset | **omission**; the paper states it, the README does not |
| README states the lock is local with no external network | **accurate**; the locker section, Lab 3, PARTS.md, and this review all state it |
| Gateway/firmware command constant | **accurate**; `gateway.py` `VENT_COMMAND_OPEN = 1` matches the firmware `0x01` open code, and `spoof.py` `VENT_COMMAND_CLOSE = 2` matches the firmware `0x02` close code |

Corrections: the README's anti-replay and key-model sections should carry the
same reboot-reset and no-per-device-rotation caveats the paper already carries.
No claim of unhackability was found, and there is no tooling command-constant
mismatch in this act.

## 8. Honest Limitations

- **Physical access wins.** A Debug Probe over SWD can read the field key from
  SRAM. The authenticated state tag detects a flipped verdict, but a probe that
  can read the key and recompute the tag defeats the design. Only OTP debug
  disable closes this.
- **Key extraction from flash.** `include/field_secrets.h` commits the passphrase
  and salt. Anyone holding the image holds the key. This is a lab convenience,
  not a deployment.
- **Single shared field key.** Both the wire key and the state-tag key derive
  from one committed secret, so a compromised device can compute tags the gateway
  accepts. There is no per-device key and no rotation in this build.
- **Replay after reboot.** `last_seq` resets to zero, so a command captured
  before a power cycle can be replayed after it. The floor is not persisted.
- **Classroom crypto profile.** Argon2id runs at `t=3 p=1 m=64` to fit SRAM; the
  state-tag nonce is deterministic in the sequence; both are teaching parameters,
  not hardening parameters.
- **The lock affects only a mock vent and a mock LCD.** The hall does not heat
  and the racks do not throttle. This is a hard scope limit, not an
  implementation detail.
- **The locker holds no real data.** The lock marker is a single byte in a
  reserved sector on the same chip that holds nothing else. There is no asset to
  release and no payment to demand.
- **The release token is a literal constant.** It is a filter, not a key. It is
  legible in the image and it is not randomized or derived.
- **The lock is inert, guarded, and breadth-limited.** It is benign,
  breadboard-bound, `SANDBOX_ONLY`-guarded, networkless, and confined to a
  reserved sector on the same chip. Its persistence is persistence against a
  firmware reflash, not against a deliberate sector erase or physical forensics.
  It demonstrates technique, not tradecraft.
- **The locker bypasses the protocol.** A local condition that overrides the
  output is an authorization-boundary, policy, and debug-lockdown problem, not a
  wire-authentication problem. Policy, gating, and debug lockdown are named as
  the real controls.
- **Anti-debug detectability is not taught to deployment depth.** The `DHCSR`
  check is deliberately simple; hardening against a determined analyst is out of
  scope.
- **Unauthenticated optical input.** Any NEC remote can send a manual purge. The
  optical surface is a documented exposure; the sealed radio path is the
  authorization path.
- **Manual purge is a single input.** It is debounced and it never bypasses
  authorization, but it is one button; a failed button or a stuck line is a
  hardware reliability problem outside the firmware's control.
- **Supply chain and sensor trust are out of scope.** The DHT11 is checksummed,
  not authenticated, and the firmware is only as trustworthy as the toolchain and
  the parts.
- **Availability is not protected by authentication.** An attacker on the band
  can jam or flood the receiver, and a local override can ignore a valid command.
- **Coverage is line coverage.** Branch coverage is not 100%, and the harness
  mocks are not the real silicon.

## 9. Conclusion

The project is internally consistent and candid: 20 owned modules, 2082
instrumented lines, 100.00% line coverage, 488 native checks passing with 0
failures, 145 native cases, and 17 passing Python tests, every cryptographic
primitive anchored to a published vector. The six gates all pass with exit 0.
Act VIII adds real availability behavior over Acts I to VII: a forced close, a
`ST:MAINT` maintenance mask, an exact-match `VAULT-RELEASE-2026` release token, a
reserved-sector `0x4C` lock marker written with the real Pico SDK flash API that
survives a firmware reflash, a re-lock on every boot, and a four-tick
re-assertion loop. It keeps the sealed and guarded vent command path, the
strictly monotonic anti-replay window, and the keyed tag over the authorization
record, all exercised end to end, and it adds a manual purge request that asks
for authorization instead of bypassing it plus a fail-open policy that opens the
vent on a lost link. Every locker behavior is asserted by a native test and every
limit is stated, including the two that matter most: the lock affects only the
mock vent and the mock LCD, and it holds no real data. The documentation is
unusually honest about the shared key, the open debug port, the inert locker, the
mock outputs, and the no-real-data scope, with minor omissions (reboot reset and
no per-device rotation) that should be folded into the README. The core lesson
holds and is stated: the wire is sealed, the verdict is tagged, the purge request
cannot bypass, and the remaining risk is the key, the probe, the local override,
and the quiet of a controller that has decided not to let the building breathe.

---

*This review is reproducible: run the six commands in section 2, the header
check in section 6, and the Python suites in section 4.*

This is Act VIII of the ten-act OPERATION COLD IRON saga. See SAGA.md.
