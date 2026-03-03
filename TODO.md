# VCU108 Porting TODO

Port Cheshire to the Xilinx VCU108 evaluation board, using the VCU118 flow as
the primary reference. The VCU108 (Virtex UltraScale XCVU095) is similar to
the VCU118 (Virtex UltraScale+ XCVU9P) but differs in FPGA family, system
clock frequency, DDR4 configuration, and pin assignments.

---

## 1. Gather VCU108 Hardware Information

Before writing any code, collect the following from the VCU108 board user guide
(UG1224) and schematic:

- [ ] Confirm FPGA part number (expected: `xcvu095-ffva2104-2-e`)
- [ ] Confirm Vivado board part string (expected: `xilinx.com:vcu108:part0:X.X`)
- [ ] Record system clock frequency and differential pin pair (expected: 300 MHz)
- [ ] Record UART RX/TX pin locations and I/O bank voltage (LVCMOS18 or other)
- [ ] Record JTAG PMOD header pin locations (TMS, TDI, TDO, TCK) and I/O standard
- [ ] Record active-high system reset pin and I/O standard
- [ ] Confirm DDR4 chip part number, data width (expect 64-bit), number of chip
      selects, and timing grade
- [ ] Confirm on-board SPI NOR flash part number and connection (CS line)
- [ ] Confirm configuration flash part string for `cfgmp` (e.g. `mt25qu01g-spi-x1_x2_x4`)

---

## 2. Create the XDC Constraint File

**File to create:** `target/xilinx/constraints/vcu108.xdc`

Model on `target/xilinx/constraints/vcu118.xdc`. Required changes:

- [ ] Set `SYS_TCK` for the VCU108 system clock period (e.g. `3.333` for
      300 MHz vs `4` for 250 MHz on VCU118)
- [ ] Update `create_clock` period and port to match VCU108 clock pins
- [ ] Keep `SOC_TCK 20.0` (50 MHz SoC clock is unchanged)
- [ ] Update `MIG_TCK` to match the DDR4 AXI clock produced by the VCU108 MIG
- [ ] Replace all UART `PACKAGE_PIN` assignments (VCU118: AW25 RX, BB21 TX)
- [ ] Replace all JTAG `PACKAGE_PIN` assignments (VCU118: J53 PMOD — N28/M30/N30/P30)
- [ ] Replace reset `PACKAGE_PIN` (VCU118: L19)
- [ ] Replace system clock `PACKAGE_PIN` pair (VCU118: E12/D12)
- [ ] Verify `IOSTANDARD` values match VCU108 bank voltages
- [ ] Keep the CDC `set_max_delay -datapath_only` constraints verbatim (paths
      are board-independent)

---

## 3. Add Board Parameters to `common.tcl`

**File to edit:** `target/xilinx/scripts/common.tcl`

- [ ] Add VCU108 entries after the `vcu118` block (lines 21–24):
  ```tcl
  # vcu108 board params
  set bpart(vcu108) "xilinx.com:vcu108:part0:X.X"   # verify version string
  set fpart(vcu108) "xcvu095-ffva2104-2-e"            # verify speed grade suffix
  set hwdev(vcu108) "xcvu095_0"
  set cfgmp(vcu108) "<spi-nor-part-string>"            # from board docs
  ```

---

## 4. Add IP Configurations to `impl_ip.tcl`

**File to edit:** `target/xilinx/scripts/impl_ip.tcl`

### 4a. Clock Wizard (`clkwiz` switch, after line 127)

- [ ] Add a `vcu108` case. The input frequency changes from 250 MHz (VCU118)
      to 300 MHz. Recalculate MMCM parameters (VCO must stay within 600–1200 MHz
      for UltraScale MMCMs):
  - `CONFIG.PRIM_IN_FREQ {300.000}`
  - `CONFIG.MMCM_CLKIN1_PERIOD {3.333}`
  - `CONFIG.MMCM_CLKFBOUT_MULT_F` — e.g. `4.000` gives VCO = 1200 MHz
  - `CONFIG.MMCM_CLKOUT0_DIVIDE_F {24.000}` → 1200/24 = 50 MHz (clk_50)
  - `CONFIG.MMCM_CLKOUT1_DIVIDE {25}` → 48 MHz (clk_48)
  - `CONFIG.MMCM_CLKOUT2_DIVIDE {60}` → 20 MHz (clk_20)
  - `CONFIG.MMCM_CLKOUT3_DIVIDE {120}` → 10 MHz (clk_10)
  - Regenerate `CLKOUT*_JITTER` and `CLKOUT*_PHASE_ERROR` values using the
    Vivado Clock Wizard GUI and copy them back

### 4b. Virtual IO (`vio` switch, around line 146)

- [ ] Add `vcu108` to the fall-through case for VCU118/VCU128 (no physical
      boot-mode switches, so the same 3-probe VIO applies):
  ```tcl
  vcu108 -
  vcu118 -
  vcu128 { ... }
  ```

### 4c. DDR4 Controller (`ddr4` switch, after line 223)

- [ ] Add a `vcu108` case. Key parameters to set (compare against VCU118 at
      lines 204–223):
  - `CONFIG.C0_DDR4_BOARD_INTERFACE` — use the board preset interface name
    from Vivado's VCU108 board file (check with `get_board_components`)
  - `CONFIG.C0.DDR4_InputClockPeriod` — period in ps of the MIG sys_clk input
  - `CONFIG.C0.DDR4_CLKOUT0_DIVIDE` — divider targeting the DDR4 data rate
  - `CONFIG.C0.DDR4_MemoryPart` — VCU108 DDR4 part (e.g. `MT40A256M16HA-075E`)
  - `CONFIG.C0.DDR4_TimePeriod` — tCK in ps per memory datasheet
  - `CONFIG.C0.DDR4_DataWidth {64}` — confirm 64-bit bus
  - `CONFIG.C0.DDR4_DataMask` — `DM_NO_DBI` or `DM_NO_DBI` per schematic
  - `CONFIG.C0.DDR4_AxiDataWidth {512}` — same as VCU118
  - `CONFIG.C0.DDR4_AxiAddressWidth` — match DDR4 capacity (31 for 2 GB)
  - `CONFIG.C0.DDR4_AxiIDWidth {8}` — same as VCU118
  - `CONFIG.C0.DDR4_CasLatency` / `CONFIG.C0.DDR4_CasWriteLatency` — from
    memory datasheet for selected speed grade

---

## 5. Update `phy_definitions.svh`

**File to edit:** `target/xilinx/src/phy_definitions.svh`

- [ ] Add a `TARGET_VCU108` block after the `TARGET_VCU118` block (line 16).
      VCU108 has the same peripheral set as VCU118 (STARTUPE3 is available on
      UltraScale as well as UltraScale+, no physical switches, VIO for boot
      control):
  ```systemverilog
  `ifdef TARGET_VCU108
    `define USE_RESET
    `define USE_JTAG
    `define USE_DDR4
    `define USE_QSPI
    `define USE_STARTUPE3
    `define USE_VIO
  `endif
  ```
- [ ] If the VCU108 JTAG PMOD supplies VDD/GND via the header, add
      `` `define USE_JTAG_VDDGND `` (see VCU128 block for reference)

---

## 6. Update `dram_wrapper_xilinx.sv`

**File to edit:** `target/xilinx/src/dram_wrapper_xilinx.sv`

- [ ] Add a `TARGET_VCU108` section alongside the `TARGET_VCU118` section. If
      the DDR4 capacity and bus width are identical to VCU118, the same
      `dram_cfg_t` values apply:
  ```systemverilog
  `elsif TARGET_VCU108
    localparam dram_cfg_t cfg = '{
      EnCdc       : 1,
      CdcLogDepth : 5,
      IdWidth     : 8,
      AddrWidth   : 31,   // adjust if VCU108 DDR4 is larger/smaller
      DataWidth   : 512,
      StrobeWidth : 64,
      MaxUniqIds  : 8,
      MaxTxns     : 24
    };
  ```
  Adjust `AddrWidth` if VCU108 DDR4 capacity differs from VCU118's 2 GB.

---

## 7. Check `cheshire_top_xilinx.sv`

**File to check:** `target/xilinx/src/cheshire_top_xilinx.sv`

- [ ] Verify that the DDR4 interface macro instantiation parameters (CS width,
      DQ width, DQS width, DM width) work for VCU108 or whether a separate
      `TARGET_VCU108` block is needed alongside the `TARGET_VCU118` block.

---

## 8. Update the Build System

**File to edit:** `target/xilinx/xilinx.mk`

- [ ] Add `vcu108` to `CHS_XILINX_BOARDS` (line 50):
  ```makefile
  CHS_XILINX_BOARDS := genesys2 vcu128 vcu118 vcu108
  ```
- [ ] Declare required IPs (after line 54):
  ```makefile
  CHS_XILINX_IPS_vcu108 := clkwiz vio ddr4
  ```

---

## 9. Create the Device Tree Overlay

**File to create:** `sw/boot/cheshire.vcu108.dts`

- [ ] Model on `sw/boot/cheshire.vcu118.dts`
- [ ] Update the SPI NOR `compatible` string if the VCU108 flash part differs
      from VCU118's `mt25qu01g`
- [ ] Adjust `spi-max-frequency` if rated differently
- [ ] Adjust `reg` (chip-select index) if the flash is on a different CS line

---

## 10. Build and Verify

- [ ] Generate IPs and build the bitstream: `make chs-xilinx-vcu108`
  - Resolve any MMCM VCO out-of-range errors by adjusting `CLKFBOUT_MULT_F`
  - Resolve any unsupported DDR4 part errors by updating the memory part string
- [ ] Check timing reports in `target/xilinx/build/vcu108.cheshire/` — all
      constraints must be met
- [ ] Program the FPGA: `make chs-xilinx-program-vcu108`
- [ ] Connect UART and verify boot output (OpenSBI → U-Boot → Linux)
- [ ] Connect JTAG via OpenOCD and confirm the CVA6 hart is reachable
- [ ] Flash the Linux image: `make chs-xilinx-flash-vcu108`
- [ ] Boot from SPI flash and verify end-to-end Linux boot

---

## Key Differences: VCU108 vs VCU118 Reference

| Parameter            | VCU118 (reference)               | VCU108 (target)                    |
|----------------------|----------------------------------|------------------------------------|
| FPGA family          | Virtex UltraScale+               | Virtex UltraScale                  |
| Vivado part          | `xcvu9p-flga2104-2L-e`           | `xcvu095-ffva2104-2-e` (verify)    |
| Board part           | `vcu118:part0:2.4`               | `vcu108:part0:X.X` (verify)        |
| System clock         | 250 MHz differential             | 300 MHz differential (verify)      |
| `MMCM_CLKIN1_PERIOD` | `4.000`                          | `3.333` (verify)                   |
| `MMCM_CLKFBOUT_MULT` | `24.000`                         | Recalculate (e.g. `4.000`)         |
| DDR4 input clock     | 4000 ps (250 MHz)                | Recalculate per system clock       |
| DDR4 memory part     | MT40A256M16LY-062E               | MT40A256M16HA-075E (verify)        |
| DDR4 tCK             | 833 ps                           | Verify from memory datasheet       |
| All PACKAGE_PINs     | See `vcu118.xdc`                 | Look up from VCU108 schematic      |
| STARTUPE3            | Yes (UltraScale+)                | Yes (UltraScale)                   |
| VIO boot control     | Yes (no physical switches)       | Yes (no physical switches)         |
