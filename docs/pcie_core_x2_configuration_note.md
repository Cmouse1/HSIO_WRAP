# PCIe Core x2 Configuration Note

## 1. Document Purpose

This document records the rationale of the current Synopsys DWC PCIe Core x2 configuration based on the CoreConsultant configuration file and PCIe Databook/Reference Book.

Configuration target:

- PCIe Gen5 x2
- Dual Mode configuration
- AXI4 interface
- Synopsys DWC PCIe Controller 6.21b

> Note: This document is organized according to the Reference Book feature categories. Parameters should be checked against the exact IP release document before final tapeout.

---

# 2. Basic Features Configuration

## CX_DEVICE_TYPE

Configured as Dual Mode.

Reason:

- Support both Root Port and Endpoint operating scenarios.
- Suitable for SoC-to-SoC PCIe interconnect where the same IP configuration may be used on both sides.

---

## CX_PCIE_VER

Configured for PCIe Gen5.

Reason:

- Enable PCIe 5.0 link capability.
- Maximum supported link speed depends on PHY capability.

---

## CX_NUM_LANE

Configured as x2.

Reason:

- Target system bandwidth requirement is x2.
- Reduces PHY, buffer and controller resource compared with x4.

---

## AMBA_INTERFACE

Configured AXI4.

Reason:

- Match SoC internal interconnect.
- Provide PCIe transaction interface through AXI bridge.

---

# 3. AXI Configuration

## MASTER_BUS_DATA_WIDTH

Configured as 256 bits.

Reason:

- Match AXI data path requirement.
- Improve PCIe payload movement efficiency.

## AXI_MSTR_CLK_FREQ

Configured as 500MHz.

Reason:

- AXI clock frequency selection affects available throughput.
- Combined with 256-bit data width provides sufficient bandwidth for PCIe Gen5 x2.

## AXI_SLAVE_CLK_FREQ

Configured as 500MHz.

Reason:

- Keep AXI master/slave domains consistent.
- Simplify CDC and timing closure.

---

# 4. PCIe Capability Configuration

## MSI/MSI-X Capability

MSI and MSI-X capability are enabled according to system interrupt requirements.

Reason:

- PCIe interrupt mechanism requires MSI/MSI-X instead of legacy INTx for high performance systems.

---

# 5. PF Configuration

## BAR Configuration

BAR0/BAR related parameters are configured according to application address mapping.

Reason:

- BAR provides PCIe memory space exposed to host software.
- BAR target mapping determines where PCIe memory requests are routed internally.

---

# 6. SR-IOV Configuration

## CX_SRIOV_ENABLE

Enabled.

Reason:

- Enable SR-IOV capability generation.
- Allows PF/VF virtualization capability.

## VF_AER_ENABLE

Enabled.

Reason:

- Allow Virtual Function error reporting capability.

Note:

- Current CoreConsultant shows internal VF count fixed as 2.
- VF resource generation is controlled by IP configuration and should be confirmed from SR-IOV related configuration parameters.

---

# 7. Power Management Configuration

## DEFAULT_PHY_PERST_ON_WARM_RESET

Enabled.

Reason:

- Ensure PHY reset behavior during warm reset matches system reset strategy.

## ASPM / L0s / L1 related options

Configured according to system low power requirement.

Supported states:

- L0
- L0s
- L1
- L3

Unsupported advanced low power states should remain disabled.

---

# 8. Data Link and Optional Features

## DL_FEATURE_EXCHANGE_ENABLE

Configuration should follow endpoint/root-port compatibility.

Reason:

- Enables Data Link Layer feature exchange mechanism.
- Allows link partners to exchange supported DL features after link initialization.

---

# 9. Memory Map / iATU

## CX_MEMORY_MAP_POSITION

Configured according to port role (USP/DSP).

Reason:

- Determines controller register view mapping.
- Important for correct DBI/application register access.

## CX_UNROLL_VIEW

Enabled.

Reason:

- Use unrolled iATU register view.
- Easier software programming model.

---

# 10. Verification Items

Before final release verify:

1. PF/VF capability generation.
2. MSI/MSI-X vector allocation.
3. BAR size and address mapping.
4. DBI access method.
5. ASPM/L1 behavior.
6. AXI outstanding capability and performance.

