# Switchboard TileLink Adapter

A Chisel project that exposes TileLink diplomacy ports as [Switchboard](https://github.com/zeroasiccorp/switchboard) queues, so C++ testbenches can drive RTL simulations over typed TileLink A/D channels.

Each TileLink transaction is packed into a 416-bit Switchboard payload — wide enough to carry a full `TLBundleA` or `TLBundleD` (opcode, param, size, source, address, mask, data, corrupt/denied) in a single beat.

## Repository layout

```
SwitchboardTLAdapter/
├── src/main/scala/
│   ├── Bundles.scala              # SBConst, SBIO, pack/unpack for TLBundleA/D
│   ├── SwitchboardTLAdapter.scala # Abstract LazyModule base class
│   ├── Top.scala                  # RTL entry points (rtlMain, lazyrtlMain)
│   └── example/
│       ├── Minimal.scala          # Raw SB packet loopback (no TL)
│       ├── TLLoopback.scala       # 1 client + 1 manager, passthrough
│       └── TLMem.scala            # 1 client + 2× TLRAM banks via xbar
├── sb_sim/
│   ├── include/                   # C++ TileLink agent library (tilelinklib, memifc)
│   ├── minimal/                   # Simulation example: raw packet exchange
│   ├── tlloopback/                # Simulation example: TL loopback
│   └── tlmem/                     # Simulation example: TL memory + ELF load
├── doc/
│   └── dependencies.md            # Verilator, switchboard, conda install guide
├── build.sc                       # Mill build definition
└── Makefile                       # RTL generation targets
```

This project uses [playground](https://github.com/morphingmachines/playground.git) as a library. Both repos must be siblings in the same directory:

```
workspace/
├── playground/
└── SwitchboardTLAdapter/
```

## Clone

```bash
git clone https://github.com/morphingmachines/SwitchboardAdapter.git
```

## Generating RTL

Uses `lazyrtl` for diplomacy-based modules and `rtl` for plain Chisel:

```bash
make lazyrtl TARGET=Loopback   # → generated_sv_dir/switchboard.TLLoopback.None/
make lazyrtl TARGET=Mem        # → generated_sv_dir/switchboard.TLMem.None/
make rtl TARGET=Minimal        # → generated_sv_dir/switchboard.Minimal.None/
```

Output Verilog and a filelist are written to `./generated_sv_dir/<module>/`. A GraphML file visualizing the TileLink diplomacy graph is also generated — open it with [yEd](https://www.yworks.com/products/yed).

## Scala console

```bash
make console
```

Load a design interactively:

```bash
scala> :load inConsole.scala
```

`inConsole.scala` loads the `TLMem` module. Query its bundle parameters:

```
scala> dut.ram0.node.in(0)._2.bundle
val res1: freechips.rocketchip.tilelink.TLBundleParameters =
  TLBundleParameters(16,32,4,1,2,List(),List(),List(),false)
```

## Simulation

Install [dependencies](./doc/dependencies.md) first, then generate the RTL for the target module.

```bash
conda activate switchboard
cd sb_sim/<example>           # minimal | tlloopback | tlmem
cmake -B build
cmake --build build --target verilator          # incremental build + run
cmake --build build --target verilator-rebuild  # force full recompile + run
cmake --build build --target clean-extra        # remove simulation artifacts
```

Enable FST waveform tracing:

```bash
cmake -B build -DTRACE=ON
cmake --build build --target verilator
```

`tlmem` additionally requires a RISC-V toolchain with `fesvr`:

```bash
export RISCV=/path/to/riscv-toolchain
cd sb_sim/tlmem && cmake -B build && cmake --build build --target verilator
```

See [sb_sim/README.md](./sb_sim/README.md) for details on the C++ TileLink agent library (`tilelinklib`, `memifc`), the 416-bit pack format, and how to adapt these examples for your own DUT.

## Continuous integration & deployment

This repo's CI/CD logic lives centrally in [shau-08/CICD](https://github.com/shau-08/CICD) — `ci.yml` and `cd.yml` here are thin callers, not the actual logic. Updates to the shared workflows propagate to this repo automatically without anything here needing to change.

### One-time setup after cloning

Git doesn't use `.githooks/` automatically — it has to be pointed there once per clone:

```bash
git config core.hooksPath .githooks
```

Skip this and `pre-push`'s lint check simply never runs — pushes go through with no local feedback, and the first sign of a problem would be CI failing remotely instead of the hook catching it before the push leaves your machine.

### What runs on a normal push or PR

`ci.yml` calls `reusable-ci.yml`, which:
- attempts a real merge against `main` first if it's a PR (fails fast and clearly on conflicts, rather than surfacing as a confusing test failure later)
- runs `make test`
- lints, auto-detecting this repo's Mill project name from `build.sc` (the object whose `millSourcePath = os.pwd`)

Locally, `.githooks/pre-push` runs that same lint check before a push leaves your machine — if it reformats anything, the push is blocked until you review and commit the changes. `.githooks/post-commit` runs the same check after every commit, non-blocking, just as an early heads-up.

### Generating and releasing RTL

RTL generation is manual and deliberate: trigger `cd.yml` via `workflow_dispatch` from the Actions tab (optionally naming a `tag_name`, or leave it blank to auto-generate one). It runs `make rtl-dispatch`, which reads **`cd.config`** to decide what to build:

```
RTL_TARGET=lazyrtl
```

`RTL_TARGET` must be `rtl` or `lazyrtl` — matching the two real Makefile targets — and `TARGET` (defaulting to `Loopback` in the `Makefile` itself) must be a value that target actually supports:

| `RTL_TARGET` | Valid `TARGET` values |
|---|---|
| `rtl` | `Minimal` |
| `lazyrtl` | `Loopback`, `Mem` |

Changing what a release build generates is a one-line edit to `cd.config`, committed and pushed like any other file — no GitHub UI, no touching `.github/workflows/*.yml`.

Once built, `generated_sv_dir/` gets packaged into a `.tar.gz` and attached to a GitHub Release (the release is aborted rather than published if no `.sv` files were actually produced).

### Keeping CI fast

`warm-verilator-cache.yml` runs weekly (and on demand) purely to keep the Verilator build cache warm, so a cold cache eviction doesn't silently make the next real CI/CD run much slower by forcing a from-source Verilator rebuild.

### Keeping the workflow files themselves honest

`validate.yml` runs `actionlint` against every file in `.github/workflows/` whenever one changes — catching YAML/syntax mistakes in the workflows before they'd otherwise fail in a real run.

### Known gaps, as of this writing

- **`playground.hash`** exists in this repo but isn't currently read by the shared `setup-toolchain` action — `playground` is checked out at its live default-branch tip regardless, not pinned to this hash. Either treat the hash as aspirational until that's wired in, or remove it so it doesn't imply a guarantee that isn't enforced.
- **This repo isn't yet notifying a super-repo on release.** `SwitchboardAdapter` is one of the real dependencies listed in `RedefineIp-master`'s `.gitmodules` (`dependencies/SwitchboardAdapter`), but its `cd.yml` doesn't set `notify_repo` — so releasing here doesn't currently trigger `RedefineIp-master` to bump its pointer and regenerate RTL the way `mmu`, `RRM`, `CR_NI`, and `network_interface` now do. Worth adding once you're ready to bring this repo into that chain.

## Chisel resources

- [Chisel Book](https://github.com/schoeberl/chisel-book)
- [Chisel Documentation](https://www.chisel-lang.org/chisel3/)
- [Chisel API](https://www.chisel-lang.org/api/latest/)
