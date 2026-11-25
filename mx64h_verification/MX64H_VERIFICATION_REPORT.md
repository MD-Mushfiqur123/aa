# MX64-H / MXISA-64 Local Verification Report

**Date:** 2026-11-25 · **Agent:** independent read-only verification agent · **Verification work directory:** `/tmp/mx64h_verification_20261125_192857/`

---

## 1. Executive summary

- The supplied bundle (`MX64H_ALL_PROJECT_DOWNLOADS.zip`, SHA-256 `4da4899b…fc7ab2` — verified exact match) is a collection of 50 phase snapshot ZIPs plus 10 demo `.elf`/`.mxb` binaries. The newest snapshot is **`mxisa64_mx64h_phase35_lsu_cache_subsystem.zip`**, which was extracted and tested as the project under test.
- All available **software-level checks PASS** (13 checks: repo structure, RTL source contracts, SHA-256 manifest 177/177, ABI, ISA table, 2 platform profiles, 8 test suites, example build + boot pack/inspect).
- **No HDL tool exists in this environment** (iverilog/vvp, verilator, yosys, questa, vivado, etc. all missing). Therefore **no lint, compile, elaboration, simulation, formal, or synthesis was run.** All 12 SystemVerilog testbenches are non-executable scaffolds with zero assertions — **NO TESTBENCH EFFECTIVELY EXISTS** for subsystem verification.
- Static RTL audit (manual read): the code quality of the phase-25…35 slices is reasonable, but **`mx64_top` is an empty shell** and the two integration composites (`mx64_frontend_dispatch`, `mx64_lsu_cache_subsystem`) are instantiated by nothing. The candidate defect with the highest risk is a **structurally possible lost same-cycle wakeup + reference leak in `issue_queue`**. Several response-attribution and invalidation-during-busy weaknesses exist in the L1D controller.
- **Confirmed:** no HDL check of any kind was performed and none can be claimed. The README/manifests are honest about this; the findings below are static source-reading conclusions and must not be treated as simulated results.

## 2. Project under test — exact identification

| Item | Value |
|---|---|
| Delivery ZIP | `/workspace/github__MD-Mushfiqur123__aa/MX64H_ALL_PROJECT_DOWNLOADS.zip` (7,371,055 bytes) |
| ZIP SHA-256 | `4da4899bbdd9b695fc29302619930b8676ba995331a73105bc100761bcfc7ab2` — **MATCHES the owner-supplied hash** |
| Outer repo | `MD-Mushfiqur123/aa` @ `bacce965ede8348dac3a24fc855398c0d247635e` ("Add files via upload"), branch `main`, worktree clean at clone time |
| Project extracted to | `/tmp/mx64h_verification_20261125_192857/project36/mxisa64_work_phase35/` (outside the repo) |
| Phase/version | README tracks Phase 35 "LSU/L1D cache subsystem integration"; root status line still reads "Architecture baseline v1.0-draft" (documentation disagrees only in flavor, not phase numbering). Three manifests shipped: PHASE33/34/35. |
| Worktree status before testing | Fresh extract, no git metadata (ZIP contains no `.git`); baseline recorded via `PHASE35_MANIFEST.sha256` (177 files, all OK) |
| Project not a git repo | Noted; git read-only inspection per §2 of the agent instructions was replaced by manifest verification (177/177 OK, re-verified after testing — tree unchanged) |
| Nested archives | 50 hashes recorded in `logs/nested_hashes.txt` (incl. all phase zips) |

## 3. Host and tool availability matrix

**OS:** Debian GNU/Linux 13 (trixie), kernel 6.18.35, x86_64.

| Tool | Version | Category |
|---|---|---|
| git 2.47.3, make 4.4.1, gcc 14.2.0, Python 3.12.14, binutils 2.44, zip/unzip | available | software checks |
| iverilog / vvp | **NOT AVAILABLE** | HDL compile/sim |
| verilator | **NOT AVAILABLE** | lint/elab/sim |
| yosys | **NOT AVAILABLE** | synthesis |
| slang / surelog / svlint / verible-verilog-lint | **NOT AVAILABLE** | parse/lint |
| questa(vsim) / vcs / xrun | **NOT AVAILABLE** | commercial sim |
| vivado / quartus_sh / nextpnr-* / openFPGALoader | **NOT AVAILABLE** | P&R / hardware |
| pytest (binary) | NOT AVAILABLE; tests are standalone scripts (executed) | — |

**Consequence:** stage categories actually run = *software/static checks* + *manual RTL source audit*. Compilation, elaboration, simulation, formal, synthesis, place-and-route, timing and hardware = **NOT AVAILABLE**. No stage was converted to PASS.

## 4. Commands executed and results

All commands run with `cwd = <project>`, `PYTHONDONTWRITEBYTECODE=1`; all outputs/logs written to `/tmp/mx64h_verification_20261125_192857/logs/` and `/tmp/…/out_example/`. Project tree verified byte-identical to the shipped ZIP after all runs (`diff -r` against pristine re-extract; manifest recheck 177/177 OK).

| # | Check | Command | Exit | Result | Type |
|---|---|---|---|---|---|
| 1 | ZIP hash | `sha256sum MX64H_ALL_PROJECT_DOWNLOADS.zip` | 0 | PASS (hash match) | evidence |
| 2 | Repo/manifest | `python3 scripts/check_repo.py` | 0 | **PASS** — 55 required files | static |
| 3 | Manifest | `sha256sum -c PHASE35_MANIFEST.sha256` | 0 | **PASS** — 177/177 OK (twice) | evidence |
| 4 | RTL contracts | `python3 scripts/check_rtl_sources.py` | 0 | **PASS** — 29 unique module decls; 222 contract checks | static |
| 5 | ABI | `python3 tools/mx64/abi_check.py` | 0 | **PASS** — 8 checks | software |
| 6 | ISA table | `python3 tools/mx64/audit_isa.py` | 0 | **PASS** — 55 opcodes, enc/dec round-trip | software |
| 7 | Platform FPGA | `python3 tools/mx64/platform_check.py platforms/fpga/template.json` | 0 | **PASS** (template only) | static |
| 8 | Platform ASIC | `python3 tools/mx64/platform_check.py platforms/asic/template.json` | 0 | **PASS** (template only) | static |
| 9–16 | assembler / architecture / elf64 / multisection / smoke / boot_image / platform / load_image tests | `python3 tools/mx64/<test>.py` | 0 (all 8) | **PASS** ×8 | software |
| 17 | Example build (redirected) | `python3 tools/mx64/mxbuild.py software/examples/{main,worker,memory}.s -o $OUT/demo.elf --entry _start --map $OUT/demo.map` | 0 | **PASS** — 8,608-byte ELF64 | software |
| 18 | Boot pack/inspect (redirected) | `python3 tools/mx64/mxboot.py pack … -o $OUT/demo.mxb --platform generic` + `inspect` | 0, 0 | **PASS** — platform=generic, entry 0x10000, payload 8,608 B | software |
| 19 | unittest discovery | `python3 -m unittest discover -s tools -t .` | 1 | **FAIL (expected)** — `tools/` not importable; tests are standalone scripts, not a package | software |
| 20 | HDL lint/compile/elab/sim/formal/synth | — | — | **NOT AVAILABLE** (no tools installed; nothing installed per rules) | HDL |
| 21 | Testbench simulation | — | — | **NOT RUN — no simulator;** all 12 `.sv` testbenches are `$display`-only scaffolds with 0 assertions | HDL |
| 22 | `make test` / `make check` | not used as make targets; individual commands above (equivalent, verified side-effect-free first) | | | — |
| 23 | `make example` / `boot-image` | **NOT RUN as-is — would modify repository** (`build/` inside tree). Executed instead with outputs redirected outside (rows 17–18). | | | — |

Refusals applied by design: `make clean`, any `git reset/checkout/clean`, package installs (`Reported`, not installed), any wave/coverage generation.

## 5. HDL results

- Lint / parse / compile / elaboration: **NOT AVAILABLE** — not run, not passed.
- Simulation: **NOT AVAILABLE**; testbenches (`verification/unit/*.sv`, 723 lines total) contain **no stimulus, no checks**; `tb_lsu_cache_subsystem.sv` literally prints "scenarios pending simulator-backed execution" (18 lines). Per-subsystem testbench existence matrix: files exist for 12 subsystems, **executable coverage exists for none** → report **NO TESTBENCH (effective)** for all critical subsystems.
- Formal: `verification/assertions/rob_properties.sv` is a 15-line near-empty template (1 property). Inline SVA `$error` assertions exist in ~10 RTL modules (good signing intent) but **were never elaborated or checked**.
- Synthesis/P&R/timing: **NOT AVAILABLE / NOT RUN**.

## 6. Subsystem integration matrix (static source evidence)

| Subsystem | Module(s) | Instantiated by | Status class |
|---|---|---|---|
| CPU top | `rtl/core/mx64_top.sv` | — (nothing) | **ABSENT — empty shell, ports not frozen, TODO MX64-RTL-001** |
| Frontend (fetch/decode/BTB) | `rtl/frontend/*` | `mx64_frontend_dispatch` | connected at subsystem level, not elaborated |
| Rename/RAT/free-list | `rtl/rename/{rename_unit,free_list}.sv` | `mx64_dispatch_controller` (via control subsystem) | connected at subsystem level |
| PRF | `rtl/rename/physical_regfile.sv` | `mx64_frontend_dispatch` | connected at subsystem level |
| ROB | `rtl/scheduler/reorder_buffer.sv` | `mx64_integer_backend` | connected at subsystem level |
| Issue queue / ALUs | `rtl/scheduler/issue_queue.sv`, `rtl/execution/*` | `integer_execution_cluster` / `mx64_integer_backend` | connected at subsystem level |
| Trap/CSR | `rtl/core/mx64_trap_controller.sv` (connected in control subsystem), `rtl/csr/mx64_csr_file.sv` (**orphan — instantiated by nothing**) | — | trap: documented+unconnected; **CSR: RTL exists but unconnected** |
| LSU/AGU/ordering/refs | `rtl/memory/{mx64_address_generation_unit,memory_ordering_unit,lsu_reference_lifecycle,mx64_lsu_pipeline}.sv` | `mx64_lsu_cache_subsystem` | connected at subsystem level |
| LSU↔L1D bridge + L1D | `rtl/memory/{mx64_lsu_cache_bridge,cache_hierarchy}.sv` | `mx64_lsu_cache_subsystem` | **connected at subsystem level (phase-35 slice), verified by source trace below** |
| Full LSU subsystem | `rtl/memory/mx64_lsu_cache_subsystem.sv` | **nothing** | subsystem top exists, not connected to any core |
| Composite core slice | `rtl/core/mx64_frontend_dispatch.sv` | **nothing** | composite exists, not connected |
| MMU/TLB | — | — | **documented only (no RTL)** |
| L1I / L2 / coherence | — | — | documented only |
| FP/SIMD | — | — | absent |
| SoC bus / memory controller | — | — | absent |
| Debug/CDC/platform | `scripts/*.bat`, docs | — | documented only |

**Phase-35 LSU↔L1D end-to-end wiring trace (confirmed by source read):** `mx64_lsu_cache_subsystem` → `mx64_lsu_pipeline` (AGU + `lsu_reference_lifecycle` + `memory_ordering_unit`) → `mx64_lsu_cache_bridge` → `cache_hierarchy` (L1D) → `mem_req_*`/`mem_rsp_*`. Ports line up; response class/slot registers latch on request accept; store commits get cache priority; store-order slot retires upon cache request acceptance. Load request metadata is latched until response in bridge (`req_offset_q/req_size_q/req_signed_q`) and cache (`req_addr_q`,`req_be_q`). Single-outstanding restriction is structurally enforced (cache `req_ready` only in `S_IDLE`), and load admission gates on it.

## 7. Physical-register lifecycle audit (static)

- **Acquire points:** IQ enqueues (unresolved srcs), LSU load base, store base+data, branch checkpoint consumers (via `branch_reference_lifecycle`), external consumer path in `mx64_integer_backend`. Initial arch regs hold 1 reference (tracker reset), replacement commits free old mapping (rename commit path).
- **Release points:** IQ flush/recover-kill or wakeup; LSU done/replay/flush; tracker reclaims on nonzero→zero transition snapshot into free list (`reclaim_bitmap`, one-cycle pulse, same-cycle alloc protection via `!free_q[j]` and alloc/free alias rejection).
- **Observed weaknesses:** (a) same-cycle enqueue+wakeup loses wakeup and leaks the just-acquired reference — see **MX64H-002**; (b) `clear_all` zeroes all counts including the 32 initial architectural references — safe only if consumers never reclaim in the same window, asserted nowhere (**MX64H-005**); (c) LSU lifecycle frees a store's data-tag reference at `store_done`; order-queue retirement (commit) path frees the ordering slot at cache-accept — the two lifetimes are separate by design, but a commit-time error returns to a slot that no longer exists (**MX64H-006**). No evidence of bypass paths (all IC-checked tags route through the single tracker instance per backend).

## 8. Memory / cache / LSU audit findings (phase-35 focus)

Verified by RTL trace (facts) and then classified:

1. L1D actually instantiated? **Yes**, inside the bridge (`mx64_lsu_cache_bridge.sv:96`), single `u_l1d`.
2. Bridge connects correct cache ports? **Yes** (all req/rsp/mem/invalidate lines traced).
3. LSU pipeline connects to bridge? **Yes** via subsystem `cache_req_*` comb mux (stores+) / loads.
4. ready/valid consistency: **OK** structurally; response-stability asserted in 3 places (cache, bridge, subsystem).
5. Request metadata latched: **Yes** (see §6).
6. Multiple outstanding transactions: **No** — enforced (`req_ready` in `S_IDLE` only; load admission additionally gated on `!store_commit_valid`).
7. PA width consistent: yes (`PADDR_W` parameterized end-to-end; default 40; tag concat reconstruction verified 40=30+4+6).
8. Byte lanes/endianness: low-aligned little-endian throughout; cache writes `lane = addr[2:0] + b`; consistent in store-hit write (`cache_hierarchy.sv` S_IDLE), refill merge (`refill_line`), and bridge shift/format (`req_offset_q*8`).
9. Sign/zero extension and subword extraction: correct in bridge `formatted_load` for 1/2/4/8 widths, signed/unsigned (case coverage complete).
10. Partial-store merge into cache lines: correct on hit (merge into line) and on miss (merge into `refill_line` before install; line marked dirty `dirty_q[saved_index_q] <= req_write_q`).
11. Youngest-older-store wins per byte: correctly implemented in `memory_ordering_unit` (per-byte best-seq selection, unresolved older stores block the load instead of forwarding).
12. Load overlapping multiple stores: handled per byte (each byte independently forwarded from the best older store; remaining bytes from cache when `load_forward_complete`).
13. Unresolved older stores (addr or data not yet resolved): conservative `older_store_unresolved` → load waits; AGU always resolves at alloc here, so only reachable if the pipeline's addr/data late-resolve path is used upstream.
14. Miss/refill/evict/writeback/error propagation: implemented; a victim-WB memory error is reported as the *current request's* error (`S_WB_WAIT → response_error_q → S_RESP`) — conservative but mis-attributing (**MX64H-004**).
15. Committed store with backpressured cache: store commit ready gated on `cache_req_ready`; order slot retention held until accepted — **OK**.
16. Store-order retire before cache accept: **No** — retire fires exactly on `store_commit_valid && store_commit_ready`.
17. Outstanding response during flush/recovery: response identity registers (`response_slot_q`, `response_is_store_q`) survive `flush_all`; invalidation/protection is delegated upstream (contract) — flagged (**MX64H-007**).
18. Response after slot reuse: since a new request cannot be accepted until the previous response is consumed, a response cannot be attributed to a reused slot structurally in the phase-35 slice; risk reappears when integration adds buffering upstream.
19. Atomics/fences/MMIO: **explicitly unsupported** for this slice (documented contracts); MMIO ranges exist only in platform templates/loader model.

## 9. Ranked defect register

### Confirmed (statically proven from source)

**MX64H-001 · BLOCKER · confidence high — No functional CPU exists; nothing is elaboratable as a core**

- `rtl/core/mx64_top.sv:1` — empty module, only clk/rst_n, "Integration ports are intentionally not frozen… This shell is not a functional CPU."
- `rtl/core/mx64_frontend_dispatch.sv` and `rtl/memory/mx64_lsu_cache_subsystem.sv` are **instantiated by no module and no testbench** (grep over `rtl/`, `verification/`).
- Trigger: any attempt at whole-CPU compile/elab/start.
- Expected: an elaboratable CPU top; observed: orphan composites.
- Impact: phase-35 slice cannot be validated end-to-end; subsystem work is unverifiable as integrated.
- Next: define `mx64_top` port contracts (SoC bus, interrupts, debug), connect `mx64_frontend_dispatch` + `mx64_lsu_cache_subsystem`, then elaborate.

**MX64H-002 · HIGH · confidence medium — issue_queue: same-cycle enqueue+wakeup loses the wakeup and leaks the acquired reference**

- `rtl/scheduler/issue_queue.sv` — the wakeup loops apply `effective_entry[e]` only `if (valid_q[e])`; the enqueue path stores `enq_uop[p]` (with `src*_ready` as-is) into the slot and issues `ref_acquire` for unresolved sources at `p_ref…` "Retain unresolved source tags…" block, but the release-on-wakeup block iterates `if (valid_q[e])` — a freshly enqueued entry has `valid_q[e]==0` in that cycle.
- Trigger: a completion on the wakeup bus lands in the same clock cycle that the dependent uop is dispatched/enqueued with `src*_ready=0`.
- Observed (RTL semantics): operand value discarded, `src1_ready` stays 0, entry never becomes ready; the acquired reference is never released by the IQ → count leak, potential permanent stall and eventual free-list starvation (a candidate root cause for the class of bugs phases 24–28 were fighting).
- Expected: same-cycle wakeup capture at enqueue (issue_uop[write_slot] = woken entry), or an upstream rule asserting this cannot occur (none documented in `MODULE_CONTRACTS.md`).
- Confidence rationale: RTL behavior is certain; whether the trigger is reachable depends on upstream dispatch timing, which cannot be simulated here.
- Next: write a directed test (IQ-001 in TEST_MATRIX already lists "Operand wakes same cycle") once a simulator is installed; or add the same-capture logic at enqueue.

**MX64H-003 · MEDIUM · confidence high — `rtl/core/reorder_buffer.sv` ships a stale orphan module `mx64_rob_placeholder`**

- Duplicate/stale artifact alongside the real `rtl/scheduler/reorder_buffer.sv`; instantiated by nothing; the public checkers only verify textual uniqueness. Next: delete or archive; risk of confusion.

**MX64H-004 · MEDIUM · confidence high — cache attributes victim-writeback memory errors to the pending load/store response**

- `rtl/memory/cache_hierarchy.sv`, `S_WB_WAIT` branch: `if (mem_rsp_error) { response_error_q <= 1'b1; … state_q <= S_RESP; }` — a writeback error on the *evicted dirty line* becomes the error response of the current request (whose line data was never fetched), and for a committed store the error surface (see MX64H-006) may be dropped.
- Impact: conservative (no data corruption) but lossy error reporting on stores; may turn a background memory fault into a spurious load fault.
- Next: NR errors should not fabricate a request-completion error for stores; define an errorful-store contract upstream.

**MX64H-005 · MEDIUM · confidence medium — reference tracker `clear_all` also clears the 32 initial architectural references and can race with same-cycle reclaim**

- `rtl/rename/physical_register_reference_tracker.sv` — `else if (clear_all) reference_count <= '0;` while `reclaim_bitmap` is combinational from pre-clear counts; a tag pulsing reclaim across the clear window could be re-freed although a consumer re-acquired logically. Only the (unverifiable) upstream sequencing protects this. Suggest retaining arch-reg references through `clear_all` and gating `reclaim_bitmap` with `!clear_all`.

**MX64H-006 · MEDIUM · confidence medium — store-commit error responses outlive the freed ordering slot**

- `rtl/memory/mx64_lsu_cache_subsystem.sv` (`store_commit_ready = … cache_req_ready`) retires the store-order entry when the L1D *accepts* the write request; if the cache later errors (refill/WB, MX64H-004), the error response identifies a lifecycle slot whose store has already been marked committed upstream. Contract
  relies on the store being architecturally committed regardless — but a *memory-error exception on a committed store* has no defined path back.
- Next: decide policy (committed stores cannot fault, or reserve the response slot against reordering) and encode it in the downstream contract doc.

**MX64H-007 · MEDIUM · confidence medium — `cache_hierarchy` silently ignores `invalidate_all` while busy**

- `cache_hierarchy.sv`: `else if (invalidate_all && state_q == S_IDLE)` — when the controller is not idle the request is dropped (a sim-only assert fires, but in synthesis it is lost). Same for a flush-during-inflight in the subsystem response path: `response_slot_q`/`response_is_store_q` are not cleared by `flush_all`.
- Next: hold a pending-invalidate register or refuse the request until idle with a visible error.

**MX64H-008 · LOW · confidence high — repo hygiene: ignored artifacts are shipped and hashed**

- 31 manifest lines cover `build/mx64h-demo.{elf,map,mxb}`, `.pytest_cache/*`, and 28 CPython-3.13 `__pycache__/*.pyc` files, even though `.gitignore` excludes them — they were force-added and shipped in the phase archive. The shipped bytecode was built with Python 3.13 (env is 3.12; no effect yet since ignores are honored).
- Next: remove `build/`, `.pytest_cache/`, `__pycache__/` from tracking and from future manifests.

**MX64H-009 · INFO · confidence high — documented limitations confirmed in source**

- Sequence-number wrap in `memory_ordering_unit` must not occur while entries are live (comment only; unenforced).
- ROB epoch wrap after 2^GEN_W allocations per slot (comment; long-latency cancellation contract).
- `mx64_csr_file` exists but is unconnected; no privilege/interrupt path reachable.
- No MMU/TLB/L1I/L2/FP/SoC RTL. Blocking single-outstanding L1D only.

### Suspected / needs simulation (not counted as failures)

- IQ `count_q` update is oddly structured (`case` mixes conditions) but was traced arithmetically correct for WIDTH=2; needs targeted simulation.
- `free_list` restore-on-flush ORs the live `reclaim_bitmap` into the restored bitmap — correct only if the tracker truly has zero refs for those tags at recovery; depends on exact release ordering of IQ/LSU/branch consumers (no simulation).
- Branch/unit recovery (`branch_checkpoint_manager`, `mx64_control_subsystem`) not line-audited to the same depth as the memory path in this pass.

## 10. Missing tests and unverified assumptions

- TEST_MATRIX.md defines 16+ scenario classes; **executed: 0** (no simulator).
- No testbench has assertions/stimulus; coverage % claims impossible.
- ISA/encoder round-trips only cover the Python reference model, not RTL decode.
- No `mx64_top` test; no load/store forwarding RTL test; no cache replay tests; no formal properties beyond 1 template property.
- Assumption not verifiable here: same-cycle wakeup/enqueue exclusion (MX64H-002); commit-time error policy (MX64H-006).

## 11. Top 10 recommended next steps (no code changed by this agent)

1. Instantiate `mx64_lsu_cache_subsystem` and `mx64_frontend_dispatch` in a real `mx64_top` (MX64H-001).
2. Install one simulator (iverilog or Verilator) and make CI run the 12 testbenches for real — currently zero HDL verification exists.
3. Turn `tb_issue_queue_recovery`, `tb_lsu_cache_subsystem`, `tb_cache_hierarchy`, `tb_lsu_cache_bridge` scaffolds into executable stimulus with $error/$fatal checks (they are empty today).
4. Fix or formally exclude the same-cycle enqueue+wakeup case (MX64H-002) — candidate permanent-stall bug.
5. Decide the committed-store memory-error policy (MX64H-004/006 combined) and assert it.
6. Make `invalidate_all` safe while busy or refuse it loudly (MX64H-007).
7. Preserve arch-reg references across `clear_all`; gate `reclaim_bitmap` with `!clear_all` (MX64H-005).
8. Remove `mx64_rob_placeholder` (MX64H-003) and drop shipped `build/`, `.pytest_cache/`, `__pycache__/` from the manifest (MX64H-008).
9. Wire `mx64_csr_file` into the control path or freeze it explicitly as out-of-scope for the current phase.
10. Add CI enforcement that the shipped SHA-256 manifest contains no ignored-file patterns.

## 12. Limitations (what could not be tested and why)

- **No HDL tool of any kind was available** and the instructions forbid installing packages — all compile/sim/formal/synthesis stages: NOT AVAILABLE, not PASS.
- The bundle's nested ZIPs were extracted and hashed but only the newest (phase 35) was fully audited; earlier phases were not re-verified beyond hashing.
- No git history inside the bundle (exported snapshot); phase provenance rests on manifests (all verified 177/177).
- Static analysis cannot prove absence of the MX64H-002 trigger; it proves the RTL has no guard for it.

## 13. Integrity

- Report SHA-256: `MX64H_VERIFICATION_REPORT_SHA256` value stamped in the evidence ZIP listing (see `REPORT_HASH.txt`).
- Project tree verified untouched: pristine re-extract `diff -r` clean; manifest recheck 177/177 OK after all runs.
- (One stray empty file named `3`, created accidentally by the agent's own shell during log-path setup, was removed immediately; verified byte-identical afterwards. No project command wrote into the tree.)

## 14. Statement

**No project source files were edited.** All logs, hashes, builds, and the final report live outside the repository under `/tmp/mx64h_verification_20261125_192857/`.
