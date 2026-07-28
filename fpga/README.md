# Vivado project for Xillybus/Zedboard control

Author: Jiacong Sun <jiacong.sun@kuleuven.be>

This repository contains the Vivado project for the LAGD chip.

To generate the Vivado project, run `make` under `fpga/`.

The generated `xillydemo.bit` should be placed under `/mnt/pl_sd/xillydemo.bit`
(mount first with `mount_pl_sd.sh`). Reboot Xillinux to load the new bitstream.

The Xillinux template is from: xillybus.com/downloads/xillinux-eval-zedboard-2.0c.zip

> Note: in the FPGA project all resets are active-high (aligning with the Vivado
> FIFO IP). The chip and DAC use active-low reset.

---

## Control interfaces

The FPGA drives **three independent controllers**, each on its own Xillybus FIFO
pair and clock domain. Two are SPI masters (the chip Quad-SPI and the DAC SPI),
driven by 32-bit command words; the third is a serial-shift configurator for the
Pomelo PLL on the 8-bit byte stream. All work the same way at the host level —
push command words/bytes into a write FIFO, optionally get a loopback word/byte on
the read FIFO — but they target different devices and protocols.

| Interface | Controller | Device | FIFOs | Mode |
|-----------|------------|--------|-------|------|
| Quad-SPI to chip | [chip_controller.sv](src/verilog/chip_controller.sv) + [quad_spi_master.sv](src/verilog/quad_spi_master.sv) | chip's on-chip `axi_spi_slave` (register/memory) | `/dev/xillybus_{write,read}_32` | mode 0, quad, bursts, 25 MHz |
| DAC + S2P | [perip_controller.sv](src/verilog/perip_controller.sv) + [dac_spi_driver.sv](src/verilog/dac_spi_driver.sv) + [s2p_driver.sv](src/verilog/s2p_driver.sv) | on-board DAC (AD8802) **and** HV9308 32-ch S2P (current-mirror bias) | `/dev/xillybus_{write,read}_32_2` | DAC: mode-3 SPI; S2P: 32-bit shift+latch; both 1 MHz |
| PLL serial config | [pll_controller.sv](src/verilog/pll_controller.sv) ([pll_command_api.sv](src/verilog/pll_command_api.sv)) | Pomelo PLL test structure (`lagd_clk_gen`) | `/dev/xillybus_{write,read}_8` | 47-bit shift + commit, 1 MHz strobe |

---

## 1. Quad-SPI to the chip

The FPGA talks to the chip's on-chip SPI slave (an ETH/PULP `axi_spi_slave`)
over a Quad-SPI bus. The path from the host (Linux on the Zynq PS) to the chip is:

```
host SW  ──Xillybus FIFOs──▶  chip_controller  ──▶  quad_spi_master  ──Quad-SPI──▶  chip (axi_spi_slave ─▶ AXI memory)
(chip_test.py)   ▲ readback         │                                                                 │
                 └─────────────── read FIFO ◀───────────────────────────────────────────────────────┘
```

- **Write FIFO** (`/dev/xillybus_write_32`): the host pushes 32-bit *command* and
  *data* words.
- **Read FIFO** (`/dev/xillybus_read_32`): the FPGA pushes readback words back to
  the host.

### Command protocol

The controller reads a stream of 32-bit words and groups them into *frames* (a
command word, then the address/data words it needs). A word is decoded as a
**command** only when the controller is expecting one (i.e. it is idle between
frames) **and** the word's top nibble is the marker `0xF`. Full encoding:
[chip_command_api.sv](src/verilog/chip_command_api.sv).

```
[31:28] marker = 0xF   [27:20] opcode   [19:16] reserved   [15:0] burst_length
```

The address and data words inside a frame are consumed *by position* — the
controller already knows they follow the command — so they may carry **any**
value (including a `0xF` top nibble). Non-command words are **never forwarded to
the chip**: the marker is only a re-sync safeguard. If the FIFO stream ever gets
misaligned, a stray word arriving where a command is expected is **discarded**
instead of being misread as a command. Because chip addresses never start with
`0xF`, a real address can never be mistaken for a command.

| opcode | operation        | FIFO frame (host → write FIFO)          | SPI cmd |
|--------|------------------|-----------------------------------------|---------|
| `0x00` | CONFIG_CLK_RST   | `[cmd]` (bit1=clk_en, bit0=rstn)        | – (local) |
| `0x01` | CONFIG_SPI_SLAVE | `[cmd]`                                 | `0x01` (enable Quad) |
| `0x02` | DATA_WRITE       | `[cmd(N)] [addr] [d0]…[d(N-1)]`          | `0x02` (write mem) |
| `0x0B` | DATA_READ        | `[cmd(N)] [addr]`  → N words read back   | `0x0B` (read mem) |
| `0xFF` | WRITEBACK_FIFO   | `[cmd]` → echoed back (loopback test)    | – (local) |

`N` = `burst_length` = number of 32-bit words. `N = 1` is a single word; `N > 1`
is a burst to consecutive word addresses (`addr`, `addr+4`, …). Up to 65535
words per frame.

### How to use it (from the host)

Helpers live in [sw/chip_test.py](../sw/chip_test.py) / [sw/lib/chipc_ISA.py](../sw/lib/chipc_ISA.py):

```python
init_spi()                       # 1) enable Quad-SPI on the slave (do this once, first)
write_mem(0x0, [0x11,0x22,0x33]) # 2) burst write 3 words to 0x0, 0x4, 0x8
read_mem(0x0, 3)                 # 3) burst read 3 words -> [0x11, 0x22, 0x33]
config_clk_rst(clk_en=1, rstn=1) # drive the chip clock-enable / reset pins
```

**`init_spi()` must run first.** The slave powers up in standard SPI; Quad mode is
only active after `CONFIG_SPI_SLAVE` writes its `reg0`.

### How the SPI works (behavior)

- **SPI mode 0** (CPOL=0, CPHA=0): clock idles low, the slave samples MOSI on the
  rising edge and changes MISO on the falling edge.
- **Quad enable**: `CONFIG_SPI_SLAVE` sends standard-SPI command `0x01` followed
  by a data byte `0x01` (sets `reg0[0]`). Until then the slave is standard SPI.
- **Reads need 33 dummy cycles** between the address and the first data word
  (slave dummy register = 32 plus one extra from the slave's RX-counter
  off-by-one). The master inserts exactly 33; this is fixed, not tunable.
- **Bursts are continuous**: one command + one start address, then the data
  streams within a single chip-select frame while the address auto-increments by
  one word. Reads/writes end when chip-select is released.
- **Flow control**: data words are streamed with backpressure. The master pauses
  the SPI clock (chip-select stays low) when the input FIFO underruns (write) or
  the output FIFO is full (read), so no data is lost — the slave is edge-driven
  and tolerates a paused clock.

---

## 2. Periphery: DAC + HV9308 S2P (shared controller)

A second controller, [perip_controller.sv](src/verilog/perip_controller.sv) on its
own FIFO pair (`/dev/xillybus_{write,read}_32_2`), drives **two devices**
multiplexed by the command opcode: the on-board DAC ([dac_spi_driver.sv](src/verilog/dac_spi_driver.sv))
and the HV9308 32-channel serial-to-parallel converter ([s2p_driver.sv](src/verilog/s2p_driver.sv))
that sets the bias resistors of the chip's analog current mirrors. Both run at
**1 MHz**. This is the standard way to host a new low-bandwidth device when no
spare Xillybus FIFO is free — add an opcode, not a stream.

Unlike the chip Quad-SPI, **each DAC access is a single self-contained 32-bit
word** — there are no follow-up address/data words and no bursts. The fields are
defined in [perip_command_api.sv](src/verilog/perip_command_api.sv):

| bits  | field     | meaning                                                       |
|-------|-----------|---------------------------------------------------------------|
| 31    | valid     | `0` = ignore the word                                         |
| 30    | writeback | `1` = echo the word to the read FIFO (loopback test), no SPI  |
| 13    | rstn      | `0` = hold the DAC in reset (`dac_rstn` low), no SPI          |
| 12    | shdn      | DAC shutdown control (active low), driven on `dac_shdn`       |
| 11:8  | addr      | 4-bit DAC register address                                    |
| 7:0   | data      | 8-bit register value                                          |

When `valid=1`, `writeback=0`, `rstn=1`, the controller runs one **12-bit
MSB-first** SPI transfer of `{addr, data}`.

Behavior:
- **SPI mode 3** (CPOL=1, CPHA=1): clock idles high, the driver changes data on
  the falling edge and the DAC samples on the rising edge.
- Chip-select stays low for the transfer and is held low for `CSB_HOLD_CYCLES`
  extra cycles afterwards to meet DAC timing.
- One transfer per word (no bursts); `dac_shdn` / `dac_rstn` are also driven
  directly from the command word.

### HV9308 S2P (opcodes on the same stream)

The HV9308 is a 32-bit shift register feeding 32 latched outputs (see
[perip_command_api.sv](src/verilog/perip_command_api.sv)):

| header | opcode | frame (host → write FIFO) | action |
|---|---|---|---|
| `0xF10…` | `S2P_WRITE` | `[cmd]` `[32-bit value]` | shift the value in MSB-first, pulse LE to latch |
| `0xF11…` | `S2P_READBACK` | `[cmd]` → 1 word | recirculating scan out of the cascade Data Out (non-destructive); returns the 32 bits |
| `0xF12…` | `S2P_OE` | `[cmd]` (bit0 = OE) | output enable; `0` blanks all outputs |

`S2P_READBACK` reads what the HV9308 silicon actually captured (via its Data Out,
wired back to the FPGA on FMC `LA13_P`), the same recirculating-scan idea as the
PLL readback. OE powers up `0` (blanked); software enables it after a write and
blanks it around any reconfiguration (`s2p_reconfigure()` does this dance). The
bit→channel/mirror mapping is the PCB designer's; the FPGA only fixes MSB-first.

> Note: the host-side DAC helpers under `sw/lib/` are legacy and predate the
> current `perip_command_api.sv` packing — when reusing the DAC path, build words
> to match the table above rather than the old helper encoding.

---

## 3. PLL serial configuration

A third controller configures the on-board **Pomelo PLL** test structure
(`lagd_clk_gen`). It reuses the 8-bit Xillybus byte stream
(`/dev/xillybus_{write,read}_8`) and its own clock, driven by
[pll_controller.sv](src/verilog/pll_controller.sv).

The PLL holds a **47-bit configuration** in two registers: a shallow shift
register clocked by `data_strb`, and a hidden register (the one the PLL actually
sees) loaded from the shallow one by `cfg_vld_strb`. Raising both strobes together
resets the registers. The FPGA drives only four wires — `clk_sel`, `data_strb`,
`data`, `cfg_vld` — at ~1 MHz; the PLL's `data_o` / `lock` outputs are observed on
the scope, not wired back.

Because the stream is byte-wide, a command is a multi-byte **frame**: a header
byte `{marker=0xF, opcode}` then an opcode-dependent payload. Encoding:
[pll_command_api.sv](src/verilog/pll_command_api.sv).

| header | opcode        | payload | action |
|--------|---------------|---------|--------|
| `0xF0` | LOAD          | 6 bytes | shift the 47-bit word in MSB-first, then commit |
| `0xF1` | LOAD_LOOPBACK | 6 bytes | LOAD + echo the 6 payload bytes back (FPGA echo) |
| `0xF2` | CLK_SEL       | 1 byte  | set the SoC clock source (0 = PLL, 1 = reference) |
| `0xF3` | RESET         | –       | pulse both strobes → reset the PLL registers |
| `0xF4` | READBACK      | – (→6 B)| scan the shallow register out of `data_o` (recirculating) → 6 bytes |
| `0xF5` | STATUS        | – (→1 B)| return 1 status byte (bit0 = PLL lock, from `pll_lock_i`) |
| `0xFF` | WRITEBACK     | –       | echo the `0xFF` header back (controller liveness) |

The 47-bit word is the value of `pll_cfg_pkg::pack_pll_cfg()`, sent little-endian
(byte0 = bits `[7:0]` … byte5 = bits `[46:40]`) and shifted into the shallow
register MSB-first.

**READBACK** verifies what the PLL silicon actually captured: it shifts the shallow
register out through `data_o` (`pad_pll_data_o` → FPGA `pll_data_i`, FMC `LA06_N`)
while recirculating each bit back in, so the register is preserved (a full 47-bit
rotation restores it — an accidental `cfg_vld` then re-commits the same value).
It returns the 47 bits as 6 little-endian bytes. This is strictly stronger than
LOAD_LOOPBACK (which only echoes what the FPGA assembled), but it taps the shallow
register, not the hidden one — so it checks the shift chain + loaded value (the
source of the commit), not the hidden register's own latches.

### How to use it (from the host)

Helpers live in [sw/tests/pll_test.py](../sw/tests/pll_test.py) /
[sw/lib/pll_driver.py](../sw/lib/pll_driver.py):

```python
pll.writeback()                                  # liveness: echoes 0xFF
word = pll.load_cfg(pdown_PD=0, pdown_VCO=0)     # build + LOAD a 47-bit config
pll.verify_load(word)                            # LOAD + FPGA echo of the 47 bits
pll.verify(word)                                 # LOAD + scan back from data_o (silicon)
pll.is_locked()                                  # read the PLL lock bit (STATUS)
pll.bring_up(lock_timeout=1.0)                   # configure -> wait for lock -> select PLL
```

> **Bring-up note:** `clk_sel` selects the SoC (RISC-V) clock and the PLL powers up
> disabled (`pdown_PD`/`pdown_VCO` = 1), so the bitstream defaults `clk_sel=1`
> (reference). Boot on the reference, configure the PLL, then **poll `STATUS` for
> lock** (`pll_lock_i` is wired back on FMC `LA10_N`) and only switch `clk_sel=0`
> once locked, with the core held in reset (the clock mux is not glitchless).
> `bring_up()` does exactly this.

---

## Simulation

Self-checking unit testbenches live under
[src/unit_tests/](src/unit_tests/) — `chip_controller` (with a behavioral ETH
slave model), `perip_controller` (AD8802 DAC model), and `pll_controller` (PLL
shift-register model):

```
make sim TB=chip_controller     # run a suite (console PASS/FAIL); TB= pll_controller / perip_controller
make sim TB=pll_controller DUMP=1   # ... and write a VCD into vivado-runs/
```

---

## Where this can (and cannot) be reused

**Reusable** as a host↔device access path:
- the chip Quad-SPI (`chip_controller` + `quad_spi_master` + command API) for
  **any chip embedding the ETH/PULP `axi_spi_slave`** (HEMAiA/Occamy-style);
- the `dac_spi_driver` as a small **mode-3, MSB-first** SPI engine for similar
  parallel-loaded DACs.

The RTL is self-contained and ports to other Xilinx FPGAs; only the host
transport (here Xillybus 32-bit FIFOs) needs swapping if you don't use
Xillinx/Zedboard.

**Not reusable as-is** for:
- a *different* on-chip SPI slave or a generic SPI flash — the Quad-SPI master
  hard-codes this slave's behavior (command bytes `0x01`/`0x02`/`0x0B`, `reg0`
  Quad-enable, the 33-cycle read dummy, MS-nibble-first quad ordering). A
  different slave needs the master adapted.
- a DAC with a different word width or SPI mode — adjust `dac_spi_driver`'s
  `SHIFT_BITS` / mode accordingly.
- non-Xillybus / non-Zedboard platforms without replacing the FIFO front-end
  (and note Xillybus requires its own IP/license).

