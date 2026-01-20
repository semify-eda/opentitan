# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenTitan is an open source silicon Root of Trust (RoT) project. This is a monolithic repository containing hardware RTL (SystemVerilog), device firmware (C/Assembly/Rust), OTBN cryptographic co-processor code (Assembly), and host tools (Python/Rust/C++).

## Build System

OpenTitan uses **Bazel** exclusively for all builds (hardware and software). Key concepts:

### Execution Environments (exec_env)

Core concept defining hardware/software configurations:
- `sim_verilator` - Verilator simulation (open source, recommended for new users)
- `sim_dv` - Commercial RTL simulators (VCS, Xcelium)
- `fpga_cw310_test_rom` - CW310 FPGA board with test ROM
- `fpga_cw310_rom_with_fake_keys` - CW310 with production-like ROM

Each exec_env includes ROM, OTP config, linker scripts, and dispatch functions.

### Custom Bazel Rules

- `opentitan_binary` - Builds C/C++ binaries for multiple exec_envs simultaneously
- `opentitan_test` - Instantiates tests across execution environments
- Rules defined in `//rules/opentitan/`

### Common Commands

```bash
# Use bazelisk wrapper for automatic version management
./bazelisk.sh <command>

# Build everything
bazel build //...

# Run all software tests (excludes broken tests)
bazel test --test_tag_filters=-broken,-dv //sw/...

# Run single test on Verilator with real-time output
bazel test --test_output=streamed //sw/device/tests:uart_smoketest_sim_verilator

# Run single test on FPGA
bazel test --test_output=streamed //sw/device/tests:uart_smoketest_fpga_cw310_test_rom

# Build Verilator simulation (takes ~1 hour first time)
bazel build //hw:verilator

# Build FPGA bitstream
bazel build //hw/bitstream/vivado:fpga_cw310

# Skip Verilator builds (faster)
bazel build --define DISABLE_VERILATOR_BUILD=true //sw/...

# Query available tests
bazel query 'tests(//sw/device/tests/...)'
bazel query 'attr(tags, verilator, tests(//sw/device/tests/...))'
bazel query 'attr(tags, cw310, tests(//sw/device/tests/...))'

# Locate built artifacts
./bazelisk.sh outquery-all <target>

# Clean build (rarely needed)
bazel clean
bazel clean --expunge  # Nuclear option, removes all caches
```

### Important Bazel Flags

- `--disk_cache=~/bazel_cache` - Share build artifacts across worktrees (add to `~/.bazelrc`)
- `--test_output=streamed` - See test output in real-time
- `--test_tag_filters=-broken,-cw310,-verilator` - Skip tests requiring optional setup
- `--define DISABLE_VERILATOR_BUILD=true` - Skip slow Verilator builds

### Test Tags

- `broken` - Known failing tests (being triaged)
- `verilator` - Requires Verilator simulation
- `cw310`, `cw310_test_rom`, `cw310_rom_with_fake_keys` - Requires CW310 FPGA board
- `dv` - Requires commercial RTL simulator
- `manual` - Excluded from wildcard builds (`//...`)
- `skip_in_ci` - Not run in CI (e.g., some ROM E2E tests)

## Repository Structure

```
hw/
├── ip/                    # Comportable IP blocks (35+ blocks: AES, UART, SPI, etc.)
├── top_earlgrey/          # Primary chip implementation (Earl Grey)
│   ├── data/              # Top-level config (Hjson)
│   ├── ip/                # Chip-specific IPs (crossbars, pinmux, PLIC)
│   └── rtl/               # Chip-level RTL
├── top_darjeeling/        # SoC-integrated RoT variant
├── dv/                    # Common DV resources
├── formal/                # Formal property verification
└── vendor/                # External vendored IPs

sw/
├── device/
│   ├── lib/
│   │   ├── dif/           # Device Interface Functions (hardware abstraction)
│   │   ├── base/          # Base utilities (MMIO, memory)
│   │   ├── runtime/       # Runtime support (interrupts, exceptions)
│   │   ├── crypto/        # Cryptographic library
│   │   └── testing/       # Test frameworks (OTTF)
│   ├── silicon_creator/
│   │   ├── rom/           # Mask ROM
│   │   └── rom_ext/       # ROM Extension (first mutable boot stage)
│   ├── tests/             # Chip-level tests
│   └── examples/          # Example programs (hello_world, etc.)
├── otbn/                  # OTBN crypto co-processor assembly code
└── host/
    ├── opentitanlib/      # Python library for host-device interaction
    └── opentitantool/     # CLI tool

util/
├── topgen.py              # Generate top-level RTL from Hjson
├── reggen.py, regtool.py  # Generate register RTL/headers/docs
├── ipgen.py               # Generate parameterized IP blocks
├── dvsim/                 # DV simulation framework
└── vendor.py              # Import external IP

doc/                       # Documentation
rules/                     # Custom Bazel rules
third_party/               # External dependencies (managed by Bazel)
signing/                   # Cryptographic signing utilities
```

## Hardware Architecture

### Comportability

OpenTitan's IP standardization framework. All IPs follow these conventions:
- Standardized interfaces: clocking, reset, interrupts, alerts
- TileLink-UL bus protocol
- Register files auto-generated from Hjson specifications
- Consistent directory structure: `data/`, `doc/`, `rtl/`, `dv/`, `fpv/`

### IP Block Structure

```
hw/ip/<block>/
├── data/          # Hjson configuration files
├── doc/           # Design spec, DV plan
├── rtl/           # SystemVerilog design
├── dv/            # UVM testbench
├── fpv/           # Formal verification (if applicable)
└── model/         # Reference C model (if applicable)
```

### Key Hardware IPs

**Security Peripherals:**
- AES, HMAC, KMAC - Cryptographic accelerators
- OTBN - Crypto co-processor (RSA, ECC)
- CSRNG, entropy_src - Random number generation
- keymgr - Key derivation and management
- otp_ctrl - One-Time Programmable memory controller
- lc_ctrl - Life cycle management

**Core Infrastructure:**
- rv_core_ibex - RISC-V Ibex processor wrapper (RV32IMCB)
- xbar_main, xbar_peri - TileLink crossbars (auto-generated)
- pinmux - Pin multiplexing (auto-generated)
- rv_plic - Platform-Level Interrupt Controller (auto-generated)

**I/O and Memory:**
- UART, SPI (host/device), I2C, USB, GPIO
- flash_ctrl, sram_ctrl, rom_ctrl

### Top-Level Configuration

Top-level designs (e.g., Earl Grey) are configured via Hjson files in `hw/top_earlgrey/data/`:
- `top_earlgrey.hjson` - Defines clocking, reset, power domains, memory map, IP instantiations
- `xbar_*.hjson` - Crossbar configurations
- Auto-generates crossbars, pinmux, interrupt controller via `topgen.py`

### Code Generation Workflow

Hardware designs use extensive code generation:
1. Edit Hjson specs in `data/` directory
2. Run generation tools (`topgen.py`, `reggen.py`, etc.)
3. Generated files placed in `rtl/autogen/` directories
4. Generated files are checked into git (CI verifies they're up-to-date)

**Important:** Never manually edit files in `autogen/` directories - they will be overwritten.

## Hardware Design Methodology

### Design Language and Style

OpenTitan hardware uses **SystemVerilog** restricted to features described in the lowRISC [Verilog Style Guide](https://github.com/lowRISC/style-guides/blob/master/VerilogCodingStyle.md). All hardware contributions must follow this style guide. Security IP must additionally follow the [Secure Hardware Design Guidelines](doc/security/implementation_guidelines/hardware/README.md).

### Development Stages

Hardware designs progress through defined stages documented in [Hardware Development Stages](doc/project_governance/development_stages.md):

1. **Specification (Sx stages)** - Feature definition, RFC process, detailed specification
2. **In Development (Dx stages)** - D0 (specification complete), D1 (functional complete), D2 (feature complete), D2S (silicon integrated), D3 (design verification complete)
3. **Signed Off (Vx stages)** - Various verification signoff levels

Each stage has specific requirements for design completeness, verification coverage, and lint/CDC cleanliness.

### Design Workflow

**New IP Block Process:**

1. **RFC (Request for Comment)** - Create proposal in Google Doc format, follow [RFC process](doc/project_governance/rfc_process.md)
2. **Detailed Specification** - Complete full spec with register definitions, interface descriptions, functional requirements
3. **Initial Skeleton** - Create basic structure:
   - Complete register definitions in Hjson format (`data/<ip>.hjson`)
   - IP top-level module instantiating auto-generated register block
   - Markdown documentation that includes register spec
   - Must be lint-clean and Comportability-compliant
4. **Full Design** - Incrementally develop full functionality through digestible pull requests
5. **Top-Level Integration** - Integrate into Earl Grey or other top-level design

**Example:** See [Pull Request #166](https://github.com/lowRISC/opentitan/pull/166) for AES skeleton design.

### Register Tool (reggen)

Registers are defined in Hjson format and auto-generated using `reggen`/`regtool`:
- Located in `hw/ip/<block>/data/<block>.hjson`
- Generates SystemVerilog RTL, C headers, documentation
- Ensures consistency between hardware and software views
- Must be regenerated after any changes to Hjson spec

```bash
# Regenerate all register blocks
make -C hw regs

# Regenerate top-level and registers
make -C hw top
```

### Linting Methodology

**Three-tier linting approach (ALL must pass for merge):**

1. **Verilator (Open Source - Semantic Lint)**
   - Required for all contributions
   - Runs in CI on every PR
   - Setup: See `doc/getting_started/setup_verilator.md`
   - Run locally: See `hw/lint/README.md`
   - Must be waived (with review) or fixed before merge

2. **Verible (Open Source - Style Lint)**
   - Required for all contributions
   - Runs in CI on every PR
   - Checks compliance with Verilog Style Guide
   - Install from: https://github.com/google/verible/
   - Must be waived (with review) or fixed before merge

3. **AscentLint (Commercial - Sign-Off)**
   - Uses "lowRISC Lint Rules" policy (LRLR-v1.0)
   - Run periodically, not on every PR
   - Results published on hardware dashboard
   - If conflict between open-source and sign-off tools, prioritize sign-off tool

**Running lint locally:**
```bash
# Run lint for a specific IP
fusesoc --cores-root=. run --target=lint lowrisc:ip:<block>:0.1
```

### Automatic Code Formatting

Verible also provides automatic formatting (experimental):
```bash
# Format SystemVerilog files in-place
util/verible-format.py --inplace --files <path to SV files>
```

**Note:** Formatter is under development. Commit changes before running, then review and potentially amend formatting.

### Assertion Methodology

SystemVerilog Assertions (SVA) are required for catching design violations:
- Assert on invalid conditions: full FIFO writes, illegal state transitions, mutex violations
- Use standardized assertion macros from `hw/formal/README.md`
- Assertions are formally verified using JasperGold (Cadence)

**Common assertion patterns:**
- `ASSERT(name, property, clk, rst)` - Standard assertion
- `ASSERT_KNOWN(name, signal, clk, rst)` - Check signal not X/Z
- `ASSERT_INIT(name, property)` - Initial state check

### Clock Domain Crossing (CDC)

**Strategy:**
1. Use proven CDC primitives from `hw/ip/prim/` (prim_fifo_async, prim_sync, etc.)
2. Enforce usage through lint and CDC checking tools
3. Sign-off CDC tool verification (tool selection in progress)

**Critical:** Never implement custom CDC logic. Always use approved primitives from `hw/ip/prim/doc/`.

### Documentation Requirements

Each IP block must have:
- **Design Specification** (`hw/ip/<block>/doc/<block>.md`) - Architecture, features, register descriptions
- **DV Plan** (`hw/ip/<block>/doc/dv/index.md`) - Verification strategy and test plan
- Use Markdown with Hugo annotations
- Register descriptions auto-included from Hjson via `reggen`
- Follow [Markdown Style Guide](doc/contributing/style_guides/markdown_usage_style.md)

```bash
# Build documentation locally
./build-docs.sh

# Generate register docs
make -C hw doc
```

### Design Verification (DV)

OpenTitan uses industry-standard UVM methodology:
- Testbenches in `hw/ip/<block>/dv/`
- Commercial simulators (VCS, Xcelium) or Verilator
- Coverage-driven verification with detailed DV plans
- See `doc/contributing/dv/README.md` for methodology

**Formal Property Verification (FPV):**
- Used for protocol compliance checking
- Testbenches in `hw/ip/<block>/fpv/`
- Uses JasperGold (Cadence)
- See `hw/formal/README.md`

### Handling IP Changes

**When modifying interrupts:**
1. Update IP Hjson and RTL
2. Update DV testbench (`dv/tb/tb.sv`) port connections
3. Regenerate top-level: `make -C hw top`
4. Regenerate documentation: `make -C hw cmdgen`
5. Update DIFs (auto-generated picks up changes, manual parts may need editing)
6. Update PLIC test: `sw/device/lib/dif/dif_rv_plic_unittest.cc`

**Regenerating everything:**
```bash
# Regenerate top-level, crossbars, registers, docs
make -C hw top_and_cmdgen
```

### Design For Test (DFT)

DFT methodology is under development. Current considerations:
- Designs target FPGA but must be silicon-ready
- Future: scan chains, BIST insertion, test mode signals
- Security challenge: DFT requires access, security requires restriction
- DFT signals (`testmode`, `scanmode`) will be added before tapeout

### FPGA vs Silicon Considerations

OpenTitan RTL is "tapeout ready" but targets FPGA emulation first:

**Differences between FPGA and Silicon:**
- Memory implementations (FPGA BRAM vs. silicon SRAM/Flash macros)
- Analog designs and I/O pads (foundry-specific)
- Physical security hardening (may be proprietary)
- Timing and power characteristics

**Compliance Equivalence:**
- Functional behavior must match between FPGA and silicon
- Compliance test suite may have platform-specific initialization
- Example: Flash controller timing differs, requiring different software init

**Goal:** Demonstrate security equivalence through comprehensive compliance testing.

## Hardware Verification

OpenTitan uses industry-strength verification methodologies aimed at tapeout-quality assurance. Verification combines **dynamic simulation** (UVM testbenches) and **Formal Property Verification** (FPV).

### Verification Stages

Hardware verification progresses through stages defined in [Hardware Development Stages](doc/project_governance/development_stages.md):

- **V0** - Testplan complete, testbench skeleton
- **V1** - Smoke tests passing, ready for integration
- **V2** - Feature complete, 80%+ coverage, stress tests
- **V3** - Verification complete, 100% coverage, sign-off ready

Each stage has specific checklist requirements in [Verification Checklists](doc/project_governance/checklist/README.md).

### Dynamic Simulation (DV)

OpenTitan uses **UVM 1.2** methodology with SystemVerilog for constrained-random functional testing.

**Sign-off tools:**
- **Synopsys VCS** - Primary simulator for coverage closure and signoff
- **Cadence Xcelium** - Secondary support (not as mature)
- **Verilator** - Open-source option (not signoff-grade yet)

**Testbench Structure:**

Each IP has a complete UVM environment in `hw/ip/<block>/dv/`:
```
dv/
├── env/           # UVM environment (agents, scoreboards, coverage)
├── tb/            # Testbench top-level, DUT instantiation
├── tests/         # Test sequences
├── cov/           # Coverage exclusion files
└── README.md      # DV document with testplan
```

**Key DV Components:**

1. **DV Base Library** (`hw/dv/sv/dv_lib/`) - Common base classes for all testbenches
2. **CIP Library** (`hw/dv/sv/cip_lib/`) - Comportable IP-specific DV infrastructure
3. **Common UVCs** - Reusable verification components:
   - TileLink agent (`hw/dv/sv/tl_agent/`)
   - UART agent, SPI agent, I2C agent, etc.
   - CSR utilities, memory models, DV utilities

**UVM RAL (Register Abstraction Layer):**
- Auto-generated from IP Hjson specs using `reggen`
- Provides SystemVerilog register model for UVM testbenches
- Located in `hw/ip/<block>/dv/env/<block>_ral_pkg.sv`

### Running DV Simulations

**dvsim** is the unified frontend for running simulations, managed via Hjson configs.

```bash
# Run all tests for an IP block with VCS
util/dvsim/dvsim.py hw/ip/uart/dv/uart_sim_cfg.hjson

# Run single test with GUI
util/dvsim/dvsim.py hw/ip/uart/dv/uart_sim_cfg.hjson \
  --select-cfgs uart_tx_rx --gui

# Run with specific seed
util/dvsim/dvsim.py hw/ip/uart/dv/uart_sim_cfg.hjson \
  --select-cfgs uart_tx_rx --seeds 123456

# Run with coverage collection
util/dvsim/dvsim.py hw/ip/uart/dv/uart_sim_cfg.hjson --cov

# Run nightly regression (100 seeds per test)
util/dvsim/dvsim.py hw/ip/uart/dv/uart_sim_cfg.hjson \
  --reseed 100

# Use Xcelium instead of VCS
util/dvsim/dvsim.py hw/ip/uart/dv/uart_sim_cfg.hjson -t xcelium
```

**Simulation configuration:**
- Main config: `hw/ip/<block>/dv/<block>_sim_cfg.hjson`
- Defines tests, build/run options, coverage settings
- Inherits from common DV config: `hw/dv/tools/dvsim/common_sim_cfg.hjson`

### Test Types and Focus Areas

**V1 Stage - Smoke Tests:**
- Basic sanity: clock, reset, simple datapath
- CSR suite: validate all register accesses (read/write/reset values)
- Prove design ready for integration

**V2 Stage - Functional Tests:**
- **Feature tests** - Exercise each design feature from spec
- **Error/security tests** - Invalid inputs, fault injection, TileLink errors
- **Stress tests** - Parallel sequences, multiple interfaces active
- **Performance tests** - Sustained bandwidth, latency requirements
- **Power tests** - Idle and high-power scenarios

**V3 Stage - Coverage Closure:**
- Unreachable coverage analysis (VCS UNR)
- 100% coverage with exclusions reviewed by designer
- All regressions passing consistently

### Formal Property Verification (FPV)

FPV mathematically proves or disproves assertions without simulation.

**Sign-off tools:**
- **Cadence JasperGold** - Primary FPV tool
- **Synopsys VC Formal** - Also supported

**FPV Testbenches:**
- Located in `hw/ip/<block>/fpv/`
- Contains SVA properties and assumptions
- Bind files connect assertions to design

**Running FPV:**

```bash
# Run FPV on specific IP
util/dvsim/dvsim.py hw/ip/uart/fpv/uart_fpv_cfg.hjson

# Run with GUI for debug
util/dvsim/dvsim.py hw/ip/uart/fpv/uart_fpv_cfg.hjson --gui

# Run all prim FPV checks
util/dvsim/dvsim.py hw/top_earlgrey/formal/top_earlgrey_fpv_prim_cfgs.hjson

# Run connectivity tests
util/dvsim/dvsim.py hw/top_earlgrey/formal/chip_conn_cfgs.hjson

# Run with VC Formal instead of JasperGold
util/dvsim/dvsim.py hw/ip/uart/fpv/uart_fpv_cfg.hjson -t vcformal
```

**Security Countermeasure Verification:**

Common security primitives verified via FPV with fault injection:
- **Sparse FSM** (`prim_sparse_fsm_flop`) - State corruption detection
- **Dual counters** (`prim_count`) - Counter mismatch detection
- **Double LFSR** (`prim_double_lfsr`) - LFSR divergence detection

Security-specific FPV configs in `hw/top_earlgrey/formal/top_earlgrey_sec_cm_fpv_cfgs.hjson`:
```bash
# Run security countermeasure FPV for a block
util/dvsim/dvsim.py \
  hw/top_earlgrey/formal/top_earlgrey_sec_cm_fpv_cfgs.hjson \
  --select-cfgs csrng_sec_cm
```

See [Security CM DV Framework](doc/contributing/dv/sec_cm_dv_framework/README.md).

### Assertion Macros

Standard assertion macros from `hw/ip/prim/rtl/prim_assert.sv`:

```systemverilog
// Basic concurrent assertion
`ASSERT(GrantOneHot_A, $onehot0(grant), clk_i, !rst_ni)

// Assert signal is known (not X)
`ASSERT_KNOWN(DataKnown_A, data_o, clk_i, !rst_ni)

// Assert property never occurs
`ASSERT_NEVER(NoOverflow_A, valid && !ready, clk_i, !rst_ni)

// Immediate assertion (for parameters)
`ASSERT_INIT(ParamValid_A, Width > 0)

// Coverpoint
`COVER(FifoFull_C, fifo_full && write_en, clk_i, !rst_ni)
```

**Naming conventions:**
- Postfix `_A` for assertions
- Postfix `_M` for assumptions (FPV)
- Postfix `_C` for coverpoints
- Postfix `_P` for properties
- Postfix `_S` for sequences

### Coverage Methodology

Two types of coverage required for signoff:

**1. Code Coverage (Implicit - Tool Generated):**

- **Line coverage** - Which RTL lines executed
- **Branch coverage** - Which if/else paths taken
- **FSM coverage** - States visited, transitions taken
- **Toggle coverage** - Signal transitions (0→1, 1→0)
  - Collected on DUT IOs and pre-verified sub-module IOs only
- **Condition coverage** - Boolean expression combinations

**2. Functional Coverage (Explicit - Manually Written):**

- Covergroups and coverpoints in SystemVerilog
- Sample interesting value combinations and corner cases
- Cross coverage - Multiple scenarios occurring together
- Written in UVM monitors/scoreboards or embedded in RTL

**Why both are needed:**
- Code coverage finds untested logic (testplan gaps)
- Functional coverage proves design works correctly (corner cases)
- 100% code coverage doesn't guarantee all behaviors tested
- 100% functional coverage doesn't guarantee all code exercised

**Coverage Commands:**

```bash
# Run regression with coverage
util/dvsim/dvsim.py hw/ip/uart/dv/uart_sim_cfg.hjson --cov

# Run UNR (unreachability analysis)
util/dvsim/dvsim.py hw/ip/uart/dv/uart_sim_cfg.hjson --cov-unr

# Results in: scratch/<run-name>/coverage/
# Open Verdi for coverage analysis:
verdi -cov -covdir scratch/<run-name>/coverage/<block>.vdb
```

**Coverage Exclusions:**

Must be reviewed and annotated with standardized tags:
- `[UNR]` - Unreachable due to design constraints
- `[NON_RTL]` - Simulation constructs (tasks, functions, initial blocks)
- `[UNSUPPORTED]` - Feature not in spec or not supported
- `[EXTERNAL]` - Covered in different testbench
- `[LOW_RISK]` - Prohibitively hard to hit, deemed low value

Exclusions stored in `hw/ip/<block>/dv/cov/<block>_cov_excl.el` and reviewed by designer.

### Coverage Closure Guidelines

**Efficient coverage collection:**
1. Enable coverage only on relevant modules at compile time
2. Collect toggle coverage **only on IOs** (DUT and sub-modules)
3. Collect all other metrics on non-pre-verified modules
4. Black-box pre-verified sub-modules (integration testing)
5. Use coverage hierarchy files (`-cm_hier` for VCS)

**Integration Testing:**
- Pre-verified sub-modules need only IO toggle coverage
- Proves correct connectivity and integration
- Rest of logic verified in dedicated testbench

### X-Propagation (Xprop)

Xprop is **enabled by default** to catch issues with unknown (X) values.

**Mode:** Pessimistic (more conservative than gate-level)

**Best practices:**
- Drive invalid signals to X when valid bit is 0
- Example: Drive TileLink data/address to X when `a_valid` = 0
- Helps find issues that gate-sim would catch but RTL-sim might miss

**Xprop issues in code:**
To disable Xprop for problematic constructs:
```systemverilog
always (* xprop_off *) @(posedge clk) begin
  // Code with multiple returns, break, continue, etc.
end
```

Or disable for entire module using `vcs_xprop_cfg_file` in sim config.

### Documentation Requirements

**1. DV Document** (`hw/ip/<block>/dv/README.md`):
- Testbench architecture, block diagram
- Verification strategy
- Agents, monitors, scoreboards, reference models
- Assertion and coverage strategy
- Links to nightly regression dashboard

**2. Testplan** (`hw/ip/<block>/data/<block>_testplan.hjson`):
- List of tests mapped to design features
- Functional coverage plan
- Machine-parseable Hjson format
- Auto-inserted into DV document as tables

**3. Regression Dashboard:**
- Nightly regression results published to web
- Maps executed tests to testplan
- Shows coverage progress

### Testbench Generation

**Initial testbench auto-generation:**

```bash
# Generate complete UVM testbench for new IP
util/uvmdvgen/uvmdvgen.py <block_name> --spec_only    # Just spec/docs
util/uvmdvgen/uvmdvgen.py <block_name> --env          # Full environment
util/uvmdvgen/uvmdvgen.py <block_name> --agent        # Just UVM agent
```

Creates: testbench, tests, DV document, testplan, sim config, and build infrastructure.

See [uvmdvgen README](util/uvmdvgen/README.md).

### Regression Types

**Sanity Regression:**
- Quick smoke tests (1-2 tests per compile-time config)
- Run daily to catch breakages from shared code changes
- Fast feedback for developers

**Nightly Regression:**
- All tests with 100 random seeds each
- Collect and merge coverage
- Publish results to dashboard
- Must complete overnight (~8 hours)

**Goal:** Consistent 100% pass rate in nightly with 100% coverage closure.

### Three-Tier Verification

**1. IP-Level DV:**
- Fine-grained stimulus control
- Constrained random testing
- Full coverage closure (no pre-verified sub-modules)
- Faster simulation, shorter development cycles
- Example: `hw/ip/uart/dv/README.md`

**2. Core (Ibex) DV:**
- RISC-V core has dedicated testbench
- Verified to full coverage closure separately
- See [Ibex DV documentation](https://ibex-core.readthedocs.io/en/latest/03_reference/verification.html)

**3. Chip-Level DV:**
- Integration testing of pre-verified IPs
- C programs running on Ibex (cross-compiled)
- TileLink agent can stub CPU for intensive testing
- See `hw/top_earlgrey/dv/` (under development)

### Key Verification Principles

1. **Quality over speed** - Industry tapeout quality required
2. **Code reuse** - Maximize shared infrastructure (DV lib, CIP lib, UVCs)
3. **Documentation-driven** - Testplan in machine-parseable format
4. **Coverage-driven** - Both code and functional coverage to 100%
5. **Constrained random** - State space exploration over directed tests
6. **Formal where applicable** - Protocol compliance, security properties
7. **Automation** - Auto-generate testbenches, RAL models, run regressions

### Common Verification Pitfalls

- **Not running lint before DV** - Catch basic bugs early with linting
- **Insufficient functional coverage** - Code coverage alone insufficient
- **Weak scoreboards** - Need independent behavior checking, not just stimulus tracking
- **Ignoring X-propagation** - Enable Xprop to catch gate-sim issues early
- **Poor exclusion documentation** - All waivers need designer review and clear rationale
- **Skipping FPV** - Formal can prove properties simulation might miss
- **Inadequate stress testing** - Parallel stimulus finds corner case bugs

## Software Architecture

### Three Software Stacks

1. **Device Software** (`sw/device/`) - Runs on Ibex RISC-V core
   - Languages: C, Assembly, Rust
   - DIFs provide hardware abstraction for each IP block
   - OTTF (OpenTitan Test Framework) for writing tests

2. **OTBN Software** (`sw/otbn/`) - Runs on OTBN crypto co-processor
   - Language: Assembly only
   - Hand-written routines for RSA, ECC operations
   - Invoked by main processor

3. **Host Software** (`sw/host/`) - Runs on development host
   - Languages: Python, Rust, C++
   - `opentitanlib` - Python library for device interaction
   - `opentitantool` - CLI tool

### Device Interface Functions (DIFs)

Hardware abstraction layer in `sw/device/lib/dif/`. Each IP block has a corresponding DIF:
- Provides type-safe C API for hardware registers
- Example: `sw/device/lib/dif/dif_uart.h` for UART IP
- Unit tests run on host (do not require hardware simulation)

### Secure Boot Flow

```
ROM (32KB, immutable Mask ROM)
  ↓ [verifies signature]
ROM_EXT (ROM Extension, in Flash)
  ↓ [verifies signature]
BL0 (Owner bootloader)
  ↓
Application firmware
```

ROM code in `sw/device/silicon_creator/rom/`, ROM_EXT in `sw/device/silicon_creator/rom_ext/`.

### Build Artifacts

Device software builds produce multiple artifacts per exec_env:
- `<target>_<device>.elf` - Linked binary with debug info
- `<target>_<device>.bin` - Raw binary
- `<target>_<device>.dis` - Disassembly with source
- `<target>_<device>.*.vmem` - Verilog memory files for simulation
- `<target>_<device>.<key>.signed.bin` - Signed binaries (for ROM_EXT/BL0)

## Development Workflows

### Python Environment Setup

Always work in a Python virtual environment:
```bash
cd $REPO_TOP
python3 -m venv .venv
source .venv/bin/activate  # Run in every new shell
pip3 install -r python-requirements.txt --require-hashes
```

### Verilator Workflow (Recommended for New Users)

1. Build and run a test:
   ```bash
   bazel test --test_output=streamed //sw/device/tests:uart_smoketest_sim_verilator
   ```
   First build takes ~1 hour to build Verilator simulation.

2. Build just the Verilator simulation:
   ```bash
   bazel build //hw:verilator
   ```

### FPGA Workflow (CW310 Board)

1. Build FPGA bitstream:
   ```bash
   bazel build //hw/bitstream/vivado:fpga_cw310
   ```

2. Run test on FPGA:
   ```bash
   bazel test --test_output=streamed //sw/device/tests:uart_smoketest_fpga_cw310_test_rom
   ```

### Design Verification (DV) Workflow

Requires commercial RTL simulator (VCS or Xcelium). See `doc/getting_started/setup_dv.md`.

### Writing Tests

Tests use the OTTF framework (`sw/device/lib/testing/test_framework/`). Basic structure:
```c
#include "sw/device/lib/testing/test_framework/ottf_main.h"

bool test_main(void) {
    // Test code here
    return true;  // Return true for pass, false for fail
}
```

Add test to `BUILD.bazel`:
```python
opentitan_test(
    name = "my_test",
    srcs = ["my_test.c"],
    exec_env = {
        "//hw/top_earlgrey:sim_verilator": None,
        "//hw/top_earlgrey:fpga_cw310_test_rom": None,
    },
    deps = [
        "//sw/device/lib/testing/test_framework:ottf_main",
        # Other dependencies
    ],
)
```

This creates two test targets: `my_test_sim_verilator` and `my_test_fpga_cw310_test_rom`.

## Code Quality

### Linting and Formatting

Run before committing (all run in CI):

```bash
# C/C++ formatting
bazel test //quality:clang_format_check --test_output=streamed
bazel run //quality:clang_format_fix

# Rust formatting
bazel test //quality:rustfmt_check --test_output=streamed
bazel run //quality:rustfmt_fix

# Bazel file formatting
bazel test //quality:buildifier_check --test_output=streamed
bazel run //quality:buildifier_fix

# License header check
bazel run //quality:license_check --test_output=streamed
```

### Style Guides

Located in `doc/contributing/style_guides/`:
- `c_cpp_coding_style.md` - C/C++ style
- `python_coding_style.md` - Python style
- `asm_coding_style.md` - Assembly style
- `hjson_usage_style.md` - Hjson configuration style

### Hardware Linting

For SystemVerilog contributions, install Verible:
```bash
# See doc/getting_started/README.md for installation
# Add to PATH: /tools/verible/<version>/bin
```

## Key Concepts

### Life Cycle States

Managed by `lc_ctrl` IP. Device progresses through states:
- RAW → TEST_UNLOCKED/TEST_LOCKED → DEV → PROD → PROD_END → RMA

Controls access to debug features, key material, OTP programming.

### Key Management Hierarchy

```
OTP Root Keys (CreatorRootKey, OwnerRootKey)
  ↓
Key Manager (keymgr)
  ↓ [derives keys based on boot measurements]
Per-stage Symmetric Keys
  ↓
Used by crypto blocks (AES, KMAC, OTBN)
```

### Entropy Generation

```
Physical Entropy Source (ring oscillators)
  ↓
entropy_src (health checks, conditioning)
  ↓
CSRNG (CTR_DRBG)
  ↓
EDN (distribution network)
  ↓
Crypto blocks (AES, KMAC, OTBN)
```

### TileLink Bus

OpenTitan uses TileLink-UL (Uncached Lightweight) protocol:
- Two crossbars in Earl Grey: `xbar_main` (high-speed), `xbar_peri` (peripherals)
- Auto-generated from `hw/top_earlgrey/data/xbar_*.hjson`
- All IPs connect via standardized TileLink interface

## Troubleshooting

### Build Failures

1. Check CI status on GitHub - ensure your commit is green
2. Try `bazel clean` for unexplained errors
3. Use `bazel clean --expunge` as last resort (removes all caches)
4. Ensure you're in the Python virtual environment: `source .venv/bin/activate`

### Verilator Out of Memory

Verilator build requires 7GB+ RAM. Close other applications during build.

### Cannot find 'ld' error (Rust builds)

Install LLVM linker: `sudo apt install lld`

Or add to `~/.bazelrc`:
```
build --@rules_rust//:extra_rustc_toolchain_dirs=/path/to/linker
```

### Git Worktrees

When using `git worktree`:
1. Enable disk cache in `~/.bazelrc`: `build --disk_cache=~/bazel_cache`
2. Run `bazel clean --expunge` before deleting worktrees

## Documentation

- **Getting Started:** `doc/getting_started/README.md`
- **Bazel Notes:** `doc/contributing/bazel_notes.md`
- **Directory Structure:** `doc/contributing/directory_structure.md`
- **Online Docs:** https://opentitan.org/book/
- **Hardware IPs:** Each IP has spec in `hw/ip/<name>/doc/`
- **Earl Grey Datasheet:** `hw/top_earlgrey/doc/datasheet.md`
- **Comportability Spec:** `doc/contributing/hw/comportability/README.md`

## Important Notes

- OpenTitan CI runs on Ubuntu 22.04 LTS
- Minimum 7GB RAM for Verilator, recommend 32GB for comfortable development
- First Verilator build takes ~1 hour; subsequent builds are much faster with caching
- Auto-generated files are checked into git - CI verifies they match generator output
- The project uses Bazel 7.x (version pinned in `.bazelversion`)
- Set `$REPO_TOP` environment variable: `export REPO_TOP=/path/to/opentitan`
