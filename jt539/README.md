# Open `jt539` compatibility module

This package provides a **source-compatible `jt539` wrapper** for public
JTCORES users of Jotego's private `modules/jt539` submodule.

It does **not** contain or reproduce the private `jt539` source. The wrapper
interface is derived from public JTCORES instantiations, especially
`cores/rungun/hdl/jtrungun_sound.v`, and uses the GPLv3 `k054539.v`
implementation from `jlrh/konami-fpga` as its backend.

## Exact compatibility interface implemented

The public JTCORE instantiation uses:

- parameter `VOLSHIFT`
- `rst`, `clk`, `cen`, `timeout`
- CPU bus: `addr[8:0]`, `we`, `rd`, `cs`, `din[7:0]`, `dout[7:0]`
- sample ROM: `rom_cs`, `rom_addr[23:0]`, `rom_data[7:0]`
- auxiliary audio: signed `aux_l[15:0]`, `aux_r[15:0]`
- output audio: signed `left[15:0]`, `right[15:0]`
- debug: `debug_bus[7:0]`, `st_dout[7:0]`

## Installation

Replace the inaccessible submodule directory with this folder:

```bash
cd /path/to/jtcores
rm -rf modules/jt539
mkdir -p modules/jt539
cp -a /path/to/jt539_open_compat/. modules/jt539/
cd modules/jt539
./fetch_open_impl.sh
```

The resulting tree should be:

```text
modules/jt539/
├── cfg/
│   └── files.yaml
├── hdl/
│   ├── jt539.v
│   ├── k054539.v
│   ├── voltab.hex
│   ├── pantab.hex
│   └── rram_zero.hex
└── fetch_open_impl.sh
```

Then:

```bash
cd /path/to/jtcores
source setprj.sh
jtcore rungun -mister -d JTFRAME_RELEASE
```

## Compatibility details

### Address bus

The K054539 loses A8 on this external bus. Public JTCORE code already supplies
the folded 9-bit form `{A[9],A[7:0]}` (or `A[8:0]` on the Premier Soccer
variant). The wrapper passes the 9-bit address straight to the open backend.

### ROM handshake

The public `jt539` interface has no `rom_ok`. In `rungun`, `cen_pcm` is a
JTFRAME 18.432 MHz enable gated against the `pcma`/`pcmb` memory buses.
The open backend has an explicit `rom_ok`, so the wrapper ties it high and
relies on the JTFRAME-gated `cen` contract.

If a different core drives `cen` continuously from an ungated clock enable,
that core may need a small ROM-ready adapter rather than this tie-high mode.

### Timer output

The open backend currently returns a constant timeout. This wrapper implements
the K054539 timer from writes to real register 0x227 (folded address 0x127) and
control register 0x22f (folded address 0x12f, bit 5).

The phase accumulator implements the MAME timing relationship for an
18.432 MHz chip enable without integer-divider drift.

### Auxiliary audio

Public `rungun` chains the second K054539 into the first through
`aux_l/aux_r`. The wrapper therefore adds the auxiliary stereo input to the
backend PCM output using signed 16-bit saturation.

This is intended as JTCORE interface compatibility. Exact analog-input
attenuation/panning can be refined later if a board proves to depend on it.

## First validation targets

1. `rungun` / Premier Soccer: compilation, timer/NMI, dual-chip sound.
2. Any single-K054539 core: CPU register POST/readback and PCM playback.
3. Compare timer edge rate for several values written to folded register
   `9'h127`.
4. Listen specifically for missing/late samples; that is the sign that a
   target needs a different ROM-ready bridge instead of the gated-`cen`
   assumption.

## Licensing

`jt539.v` in this compatibility package is newly written glue.
The fetched backend files retain the licensing and notices of
`jlrh/konami-fpga` (GPLv3). Keep those notices intact.
