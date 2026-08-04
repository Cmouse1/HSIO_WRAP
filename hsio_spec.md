# 技术规格书：HDD HSIO PCIe/Ethernet Subsystem

**版本**：1.70
**状态**：草稿

---

## 目录

- [Figure List](#figure-list)
- [Table List](#table-list)
- [1. Introduction](#1-introduction)
- [2. Feature List](#2-feature-list)
  - [2.1 Subsystem Feature](#21-subsystem-feature)
  - [2.2 PCIe Controller Feature](#22-pcie-controller-feature)
  - [2.3 PHY Bifurcation Feature](#23-phy-bifurcation-feature)
- [3. Functional Description](#3-functional-description)
  - [3.1 Architecture](#31-architecture)
  - [3.2 Harden Division](#32-harden-division)
  - [3.3 Lane Assignment](#33-lane-assignment)
  - [3.4 Data Path](#34-data-path)
  - [3.5 Matrix](#35-matrix)
  - [3.6 Clock and Reset](#36-clock-and-reset)
  - [3.7 Sub-IPs](#37-sub-ips)
  - [3.8 IO Interfaces](#38-io-interfaces)
  - [3.9 Interrupt](#39-interrupt)
  - [3.10 Address Mapping](#310-address-mapping)
- [4. PCIe Controller Configuration](#4-pcie-controller-configuration)
  - [4.1 Controller Instance](#41-controller-instance)
  - [4.2 HDMA / SR-IOV Resource Profile](#42-hdma--sr-iov-resource-profile)
  - [4.3 SR-IOV / HDMA Software Model](#43-sr-iov--hdma-software-model)
  - [4.4 AXI Outstanding Configuration](#44-axi-outstanding-configuration)
  - [4.5 AXI Link-Down Handling](#45-axi-link-down-handling)
  - [4.6 PCIe Tag Configuration](#46-pcie-tag-configuration)
  - [4.7 Flow-Control Credit Configuration](#47-flow-control-credit-configuration)
  - [4.8 Synopsys Define / Parameter Configuration](#48-synopsys-define--parameter-configuration)
  - [4.9 X2/X1 Profile Configuration Considerations](#49-x2x1-profile-configuration-considerations)
- [5. Block Description](#5-block-description)
  - [5.1 sys_pcie](#51-sys_pcie)
  - [5.2 sys_hsio_hybrid](#52-sys_hsio_hybrid)
  - [5.3 NOC / Bus](#53-noc--bus)
  - [5.4 APR](#54-apr)
  - [5.5 CKM](#55-ckm)
  - [5.6 AHB Backdoor](#56-ahb-backdoor)
  - [5.7 SYS-ETH](#57-sys-eth)
- [6. Open Items](#6-open-items)
- [7. Change History](#7-change-history)

## Figure List

- [图 1 顶层架构图](#fig-top-arch)
- [图 2 Harden 切分图](#fig-module-division)
- [图 3 Clock 结构图](#fig-clock-structure)
- [图 4 Reset 结构图](#fig-reset-structure)
- [图 5 sys_pcie 结构图](#fig-sys-pcie)

## Table List

- [表 1 Controller 与 Lane 资源汇总](#tbl-controller-lane-summary)
- [表 2 sys_hsio_hybrid Lane 分配](#tbl-hybrid-lane)
- [表 3 AXI Matrix 位宽规划](#tbl-axi-matrix)
- [表 4 PCIe Controller Instance 配置](#tbl-pcie-controller-instance)
- [表 5 HDMA / SR-IOV 差异化资源配置](#tbl-hdma-sriov-profile)
- [表 6 SR-IOV / HDMA 软件模型](#tbl-sriov-hdma-sw-model)
- [表 7 AXI Outstanding 配置](#tbl-axi-outstanding)
- [表 8 PCIe Tag 配置](#tbl-pcie-tag)
- [表 9 Flow-Control Credit 配置策略](#tbl-credit-config)
- [表 10 RTT=1000ns Credit 校核目标](#tbl-credit-estimation)
- [表 11 Synopsys Define / Parameter 配置](#tbl-snps-params)
- [表 12 X2/X1 Profile 补充参数与集成考量](#tbl-x2-coreconsultant-analysis)
- [表 13 Ethernet Controller 配置](#tbl-eth-config)

---

## 1. Introduction

`sys_pcie_eth_pwr_wrap` 是 HSIO PCIe/Ethernet 多协议端口聚合子系统。子系统包含 2 个 PCIe Controller、2 个 10G Ethernet Controller、1 个 X4 32G-PHY，以及连接 SNOC/MNOC 的 HSIO 片上互联。

本规格定义当前项目对 Synopsys PCIe Gen5 Dual Mode Controller、PHY bifurcation、HDMA、SR-IOV、AXI 接口、Ethernet DMA 接口和公共控制模块的配置需求。未最终冻结的条目统一标记为 `[TBD]`。

---

## 2. Feature List

### 2.1 Subsystem Feature

- 子系统包含 2 个独立 PCIe Controller：1 个 max X2、1 个 max X1。
- 子系统包含 2 个 10G Ethernet Controller，固定使用 `PHY_HYBRID` Lane2/Lane3。
- 子系统包含 1 个 X4 32G-PHY：`PHY_HYBRID`。
- PCIe 与 Ethernet Controller 共享 `hsio_bus`、`hsio_sub_bus` 以及 NoC/DDR 资源。
- PCIe Controller 与 PHY lane 通过启动期静态 bifurcation 配置连接，运行期间不进行 lane 动态重分配。
- PCIe Controller 支持 EP/RC dual mode，role 在启动阶段静态确定，运行期间不进行 EP/RC 动态切换。

### 2.2 PCIe Controller Feature

以下条目适用于全部 2 个 PCIe Controller。

- **Controller IP**：Synopsys PCIe Gen5 Dual Mode Controller。
- **PCIe 速率**：支持 Gen1 / Gen2 / Gen3 / Gen4 / Gen5，向下兼容，链路速率自动协商。
- **Link Width**：按 Controller instance 支持 X2 或 X1。
- **Role**：支持 Endpoint 和 Root Complex/Root Port，启动阶段静态选择。
- **HDMA**：使用 Controller 内部 HDMA；X2 profile 配置 4 个 write channel 和 4 个 read channel，X1 profile 配置 2 个 write channel 和 2 个 read channel。
- **SR-IOV**：支持并启用；X2/X1 profile 均配置 1 PF、4 VF。
- **MSI/MSI-X**：PF 和 VF 均支持 MSI 与 MSI-X。
- **Interrupt Resource**：X2/X1 profile 的 PF0 均支持最多 32 个 MSI messages 和 32-entry MSI-X Table；VF MSI/MSI-X 资源保持一致。
- **Legacy Interrupt**：支持。
- **ARI**：支持并启用，用于 SR-IOV function number 扩展。
- **iATU**：支持 internal iATU；X2/X1 profile 均配置 8 个 outbound region 和 8 个 inbound region。
- **FLR**：支持 Function Level Reset。
- **Partial Reset**：支持各 Controller 独立 partial reset。
- **Hot Reset**：支持软件控制延迟恢复机制。
- **AER**：PF 和 VF 均支持 Advanced Error Reporting；X2/X1 profile 的 4 个 VF 均共用 PF0 的 VF Header Log 资源。
- **RAS DP**：支持 Data Protect 功能。
- **ASPM**：仅支持 L0s 和 L1；不支持 L1.1、L1.2、L2。
- **PCI-PM**：支持 PCI Power Management D-state；链路可进入 L1。
- **Separate Refclk**：支持 SRNS（Separate Refclk No Spread）和 SRIS（Separate Refclk Independent SSC）。
- **Virtual Channel**：支持 1 个 VC。
- **Max Payload Size**：配置为 256B。
- **PIPE 接口**：支持 PIPE 4.4.1。
- **AXI Manager 接口**：`256bit @ 500MHz`，作为 HDMA 主通道。
- **AXI Subordinate 接口**：`64bit @ 500MHz`，作为 CPU/软件通过寄存器发起非 DMA MMIO/config 访问的从通道，并支持 outbound NP read 完成功能。
- **ATS/PASID/PRS/TLP Prefix/TPH**：不支持。
- **Hot-Plug**：不支持。
- **Resizable BAR**：X2/X1 profile 的 PF BAR1、BAR2/3 和 BAR4/5 使用 Resizable BAR sizing scheme。
- **PTM**：X2/X1 profile 均启用 Precision Time Measurement。
- **No Snoop**：X2/X1 profile 均对 PF0 声明支持 No Snoop；事务属性到 ACE-Lite/NoC 的一致性处理必须由系统集成验证。
- **PF0 BAR0 Internal Window**：EP mode 下，X2/X1 profile 均通过 PF0 BAR0 暴露 Controller PL、unrolled iATU、unrolled HDMA 和 integrated MSI-X Table/PBA 窗口。
- **IDE / DPC / eDPC / NPEM / DPA / OBFF / Atomic Operation**：启用策略为 `[TBD]`。
- **功能安全**：支持，具体机制为 `[TBD]`。

### 2.3 PHY Bifurcation Feature

- `PHY_HYBRID` Lane0/Lane1 支持 PCIe X2 或 PCIe X1+X1；Lane2/Lane3 固定用于 10G Ethernet。
- PCIe 与 Ethernet 支持跨协议共享 `PHY_HYBRID`，PHY 内部使用不同 PLL。
- Lane reversal、lane numbering 和 PIPE port mapping 不受限制。
- `PHY_HYBRID` 具有 1 路 refclk，使用外部晶振输入。
- `PHY_HYBRID` 具有 1 路 PERST。
- 每个 lane 具有独立 reset，lane reset 来自对应 Controller。
- 每个 bifurcated PCIe link 具有独立 LTSSM。
- Controller 支持小于最大 lane 数运行并释放未用 lane；释放 lane 可被其他 Controller 使用。

---

## 3. Functional Description

### 3.1 Architecture

`sys_pcie_eth_pwr_wrap` 采用 PHY-centric 架构，以 `PHY_HYBRID` 一个 X4 32G-PHY 为核心，通过启动期静态 bifurcation 将 PHY lane 分配给 PCIe 和 Ethernet Controller。

<a id="fig-top-arch"></a>**图 1 顶层架构图**

![图 1 顶层架构图](./images/top_architecture.png)

子系统的主要组成如下：

- **PCIe Controller**：2 个独立 instance，均位于 `sys_hsio_hybrid`。
- **Ethernet Controller**：2 个 10G Ethernet Controller，位于 `sys_hsio_hybrid`，固定使用 `PHY_HYBRID` Lane2/Lane3。
- **PHY**：`PHY_HYBRID` 承载 PCIe + Ethernet 混合 lane。
- **Bus / NoC**：`hsio_bus` 汇聚 SNOC 下行访问、PCIe/Ethernet DMA 上行访问和 PHY SRAM 后门访问；`hsio_sub_bus` 将上行访问拆分至 MNOC 与 SNOC。
- **公共控制模块**：包含 `sys_ctrl`、CRG、`ecc_aggr`、CKM、RSC、XAF、APR 和 SRAM。

### 3.2 Harden Division

`sys_pcie_eth_wrap` 的 harden 切分以 `sys_pcie`、`sys_hsio_hybrid`、HSIO bus/common control 以及 PHY/analog boundary 为主要边界。Harden 切分图用于定义各 harden partition 的模块归属、跨 harden 接口和后端集成边界。

<a id="fig-module-division"></a>**图 2 Harden 切分图**

![图 2 Harden 切分图](./images/module_division.png)

### 3.3 Lane Assignment

<a id="tbl-controller-lane-summary"></a>
**表 1 Controller 与 Lane 资源汇总**

| Controller | 所属 Block | Max Lane | Active Mode | PHY/Lane | Role |
| :--- | :--- | :--- | :--- | :--- | :--- |
| C0 | `sys_hsio_hybrid` | X2 | X2 或 X1 | `PHY_HYBRID` Lane0~1 或 Lane0 | EP/RC boot-time dual mode |
| C1 | `sys_hsio_hybrid` | X1 | X1 | `PHY_HYBRID` Lane1 | EP/RC boot-time dual mode |
| ETH0 | `sys_hsio_hybrid` | X1 | X1 | `PHY_HYBRID` Lane2 | 10G/5G Ethernet |
| ETH1 | `sys_hsio_hybrid` | X1 | X1 | `PHY_HYBRID` Lane3 | 10G/5G Ethernet |

`PHY_HYBRID` 支持以下组合：

- C0 X2 使用 Lane0/Lane1，同时 ETH0/ETH1 使用 Lane2/Lane3。
- C0 X1 使用 Lane0，C1 X1 使用 Lane1，同时 ETH0/ETH1 使用 Lane2/Lane3。

<a id="tbl-hybrid-lane"></a>
**表 2 sys_hsio_hybrid Lane 分配**

| Controller | Lane 分配 | AXI 位宽 | AXI Manager BDP Outstanding | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| C0 PCIe X2 Controller | Lane0~Lane1 或 Lane0 | AXI 256-bit | 31 | X2 active mode 使用 Lane0/Lane1；X1 active mode 使用 Lane0 |
| C1 PCIe X1 Controller | Lane1 | AXI 256-bit | 16 | 在 X1+X1 PCIe 拆分模式下使用 |
| ETH0 | Lane2 | AXI 128-bit | 10 | 10G/5G Ethernet |
| ETH1 | Lane3 | AXI 128-bit | 10 | 10G/5G Ethernet |

### 3.4 Data Path

子系统包含以下主要事务路径：

- **SNOC 下行访问**：SNOC 事务通过 SNOC TNIU 进入 `hsio_bus`，访问 PCIe、Ethernet、PHY、公共控制寄存器和 SRAM 相关资源。
- **PCIe HDMA 上行访问**：PCIe Controller 通过 AXI Manager 发起 HDMA 事务，经 `hsio_bus` 进入 `hsio_sub_bus`，并根据地址路由到 MNOC 或 SNOC。
- **PCIe AXI Subordinate 访问**：CPU/软件通过 AXI Subordinate 访问 Controller CSR，并发起非 DMA MMIO/config 访问；该接口支持 outbound NP read 完成功能，不作为高吞吐数据通道。
- **Ethernet DMA 上行访问**：Ethernet manager 作为 Ethernet DMA 口，以 AXI 128-bit 接入 `hsio_bus`，并根据地址路由到 MNOC 或 SNOC。
- **PHY SRAM 后门访问**：AHB 通路访问 PHY 内部 SRAM，用于 PHY 启动前 load firmware。

### 3.5 Matrix

NOC 总线由 `hsio_bus` 和 `hsio_sub_bus` 组成。主要接口位宽规划如下。

<a id="tbl-axi-matrix"></a>
**表 3 AXI Matrix 位宽规划**

| 路径/接口 | 位宽 | 说明 |
| :--- | :--- | :--- |
| SNOC TNIU -> `hsio_bus` | AXI 64-bit | SNOC 下行访问 HSIO 子系统 |
| `hsio_bus` 主干 | AXI 256-bit | HSIO 内部主要数据通路 |
| `hsio_bus` -> `hsio_sub_bus` | AXI 256-bit | HSIO 上行进入 `hsio_sub_bus` |
| `hsio_sub_bus` -> MNOC INIU | AXI 256-bit | DDR 地址空间上行访问 |
| `hsio_sub_bus` -> SNOC INIU | AXI 64-bit | 非 DDR 地址空间上行访问 |
| PCIe Controller AXI Manager | AXI 256-bit | PCIe HDMA 主通道 |
| PCIe Controller AXI Subordinate | AXI 64-bit | CPU/软件非 DMA MMIO/config 访问从通道 |
| Ethernet manager | AXI 128-bit | Ethernet DMA 通道 |

子系统内 AXI 接口时钟统一为 500MHz。

`hsio_bus` 包含以下通路：

- `snoc2hsio`：SNOC 到 HSIO 的下行通路，接口位宽为 AXI 64-bit。
- `hsio2ring`：HSIO 到 ring/sub-ring/`hsio_sub_bus` 的上行通路，主干位宽为 AXI 256-bit。所有 PCIe Controller 和 Ethernet Controller 的上行 DMA 通路接入 `hsio_bus` 同一个 slave-side 汇聚口，并从 `hsio_bus` 同一个 master-side 出口进入 `hsio_sub_bus`，再路由至 MNOC 或 SNOC。
- AHB 后门访问通路：访问 PHY 内部 SRAM，用于 PHY firmware load。
- 异步边界：PCIe、Ethernet、公共控制模块与 `hsio_bus` 之间存在多个 CDC 边界。

`hsio_sub_bus` 将 `hsio2ring` 上行数据拆分为以下两类：

- `hsio2mnoc`：路由 DDR 地址空间，接口位宽为 AXI 256-bit。
- `hsio2snoc`：路由非 DDR 地址空间，接口位宽为 AXI 64-bit。

TBU 的 ADB 接口连接 TCU；ITS 的 AXI-Stream 接口连接 GIC。

### 3.6 Clock and Reset

<a id="fig-clock-structure"></a>**图 3 Clock 结构图**

![图 3 Clock 结构图](./images/clock_diagram.png)

<a id="fig-reset-structure"></a>**图 4 Reset 结构图**

![图 4 Reset 结构图](./images/reset_diagram.png)

时钟与复位结构定义如下：

- `PHY_HYBRID` 使用外部晶振输入 refclk。
- PERST 按 PHY 粒度配置，`PHY_HYBRID` 具有 1 路 PERST。
- Lane reset 按 lane 粒度配置，每个 lane 接收来自对应 Controller 的独立 reset。
- PCIe LTSSM 按 Controller/link 独立运行。
- `PHY_HYBRID` 内 PCIe 与 Ethernet 使用 PHY 内部不同 PLL。
- CKM 监测 PHY PLL 输出时钟稳定性，并向复位/初始化流程提供时钟稳定状态。
- APR 提供 PCIe 和 Ethernet partial reset 控制，实现单 Controller 独立复位和异常隔离。
- PCIe 支持 Common Clock、SRNS 和 SRIS；各模式的参考时钟分发、时钟精度约束及冷复位/热复位/partial reset 释放顺序为 `[TBD]`。

### 3.7 Sub-IPs

子系统按功能划分为以下 Sub-IP：

- **PCIe-Core**：Synopsys PCIe Gen5 Dual Mode Controller，共 2 个独立 instance。
- **32g-PHY**：`PHY_HYBRID` 一个 X4 PHY。
- **AHB-BUS**：PHY SRAM 后门访问通路。
- **APR**：PCIe/Ethernet partial reset 和异常隔离控制。
- **XAF**：SNOC 下行 AXI 64-bit 访问路径上的控制/访问模块，寄存器窗口和异常处理为 `[TBD]`。
- **ecc_aggr**：聚合 HSIO 子系统内部 ECC 或数据保护相关错误，错误源、上报方式和中断映射为 `[TBD]`。
- **CKM**：监测 PHY PLL 输出时钟稳定性。
- **SYS-ETH**：2 个 10G Ethernet Controller。

### 3.8 IO Interfaces

IO 接口信号定义为 `[TBD]`。

### 3.9 Interrupt

中断映射为 `[TBD]`。

### 3.10 Address Mapping

地址映射表维护在 [address_mapping.xlsx](./tables/address_mapping.xlsx)。

EP mode 下，PCIe Core 内部 PF0 BAR0 子窗口采用表 11 的固定配置。`PL_OFFSET_BAR=0` 不等同于 PL register 从 byte offset `0` 开始：在 `DISABLE_CFG_MAP_PL_REG=0` 的 Reference 默认配置下，PL register 的有效映射范围固定为 byte offset `0x700~0x8ff`。iATU、HDMA 和 MSI-X Table/PBA 使用各自独立的 offset，最终 PF0 BAR0 aperture、target map 和系统地址由地址映射表统一冻结。RC/RP mode 的本地 DBI/PL/iATU/HDMA 访问继续使用 SoC 集成侧寄存器路径，不依赖 PF BAR enumeration。

---

## 4. PCIe Controller Configuration

### 4.1 Controller Instance

PCIe-Core 采用 Synopsys PCIe Gen5 Controller，配置为 2 个独立 Controller instance。

<a id="tbl-pcie-controller-instance"></a>
**表 4 PCIe Controller Instance 配置**

| Controller | Profile | Max Lane | Active Mode | PHY/Lane | AXI Manager | AXI Subordinate | AXI Manager BDP Outstanding | `CC_MAX_MSTR_TAGS_AXI` |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| C0 | X2 profile | X2 | X2 或 X1 | `PHY_HYBRID` Lane0~1 或 Lane0 | AXI 256-bit @ 500MHz | AXI 64-bit @ 500MHz | 31 | 32 |
| C1 | X1 profile | X1 | X1 | `PHY_HYBRID` Lane1 | AXI 256-bit @ 500MHz | AXI 64-bit @ 500MHz | 16 | 16 |

AXI Manager BDP outstanding 按对应 PCIe link 的有效带宽、单 request 有效数据 256B 和 RTT=1000ns 计算。AXI Manager `256bit @ 500MHz` 的理论带宽为 16GB/s，高于 Gen5 x2/x1 PCIe link 的有效带宽，因此当前 outstanding 仍由 PCIe link BDP 决定，X2/X1 profile 分别为 31/16。该值用于规划本地 NoC/DDR 侧 HDMA request 并发能力，不等同于 PCIe tag 数量。

### 4.2 HDMA / SR-IOV Resource Profile

<a id="pcie-core-hdma-sriov-profile"></a>
<a id="tbl-hdma-sriov-profile"></a>
**表 5 HDMA / SR-IOV 差异化资源配置**

| Profile | Controller | HDMA Channel | PF/VF 配置 | Interrupt 配置 | iATU Region 配置 | 配置影响 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| X2 profile | C0 | 8 channels：4 write + 4 read | 1 PF；PF0 4 VF | PF/VF 支持 MSI/MSI-X | OB 8；IB 8 | 增加 HDMA 并行队列和地址转换窗口，覆盖 X2 Gen5 吞吐场景 |
| X1 profile | C1 | 4 channels：2 write + 2 read | 1 PF；PF0 4 VF | PF/VF 支持 MSI/MSI-X；PF MSI 最多 32 messages；MSI-X 32 entries | OB 8；IB 8 | 增加双向并行 channel 和地址转换窗口；16 个 HDMA read tags 可覆盖 256B/1us 的 Gen5 x1 BDP |

Controller 在低于最大 lane 数运行时仍使用所属 profile，不进行运行期资源裁剪。

### 4.3 SR-IOV / HDMA Software Model

SR-IOV 作为 EP mode 功能启用。RC mode 下本端不启用 VF；RC mode 用于枚举、配置和访问下游 SR-IOV EP 的 PF/VF。

X2 和 X1 profile 均配置 1 个 PF，PF0 配置 4 个 VF，`CX_NVFUNC` 派生为 4。两个 profile 均使用 internal VF、static allocation，`CX_VF_STRIDE_ALWAYS_ONE=1`；其余未显式覆盖的 VF allocation 参数使用 Reference/CoreConsultant 默认值。

SR-IOV enable 与 MSI/MSI-X enable 不是同一个功能开关。PF 和 VF 均支持 MSI 与 MSI-X。X1 与 X2 的 PF/VF MSI-X capability、32-entry Table 规模和 VF Table 位置相同；两个 profile 的 `DEFAULT_MULTI_MSI_CAPABLE_0=0x5`，PF0 最多支持 32 个 MSI messages。

Databook 规定，EP mode 下 HDMA 寄存器窗口只能分配给一个 PF，不能分配给 VF。该限制不仅适用于 HDMA 全局寄存器，也适用于 channel context、status 和原生 HDMA doorbell；因此 VF 不能直接配置或启动 HDMA channel。HDMA 的部分 doorbell、interrupt 和 MSI 资源由多个 channel 共用，Core 本身不提供满足各 System Image 隔离要求的完整 HDMA 虚拟化机制。

`HDMA_FUNC_NUM_OFF_[WR|RD]CH_i` 用于把一个 HDMA channel 关联到指定 PF/VF。Core 使用其中的 `PF`、`VF_EN` 和 `VF` 字段形成该 channel 生成事务的 Requester ID，以及相关 Completion 的 Completer ID。该寄存器只定义 PCIe function 上下文，不会把 HDMA 寄存器窗口分配给 VF，也不提供 channel 访问权限隔离。`VF_EN=1` 时，`VF` 字段编码从 0 开始；在本项目 PF0 配置 4 个 VF 的条件下，`VF=0..3` 分别对应 PF0 的 VF1..VF4。

本项目由 PF0 统一拥有 HDMA 寄存器。VF 通过项目自定义的 VF BAR queue/doorbell/status window 提交 DMA 请求；该 window 和请求仲裁属于 SoC DMA virtualization logic，不是 PCIe Core 原生 HDMA 接口。PF software 或 virtualization logic 必须校验 VF 请求、分配空闲 channel、配置 channel context 和 `HDMA_FUNC_NUM_OFF_[WR|RD]CH_i`，最后写原生 HDMA doorbell 启动传输。

X2 profile 配置 4 个 write channel 和 4 个 read channel；X1 profile 使用 Reference 默认的 2 个 write channel 和 2 个 read channel。Write channel 用于本地 application memory 到远端 PCIe link partner 的传输，read channel 用于远端 PCIe link partner 到本地 application memory 的传输。两个 profile 的每 channel HDMA linked-list descriptor prefetch queue depth 均为 8，因此 Databook 定义的每方向 overlay RAM 深度分别为 X2 `4*8=32 entries`、X1 `2*8=16 entries`。`CC_NUM_DMA_RD_TAG` 是同一 profile 全部 read channels 共享的 PCIe MRd tag reserve，X2/X1 分别为 32/16，并非每个 read channel 独占该数量。HDMA 软件初始化时必须为同一方向的全部 channel 一致设置 `PF_DEPTH`，且不得在其余 channel 寄存器开始配置后再修改。

X2/X1 profile 均使用 `CX_DMAREG_CHADDR_SPACE=4`，即各 HDMA channel 配置寄存器窗口按 4KB 间隔排列。SoC 地址表和软件寄存器定义必须按该间隔展开 channel window。

<a id="tbl-sriov-hdma-sw-model"></a>
**表 6 SR-IOV / HDMA 软件模型**

| 请求来源 | 软件可见入口 | 原生 HDMA 寄存器访问 | Channel 配置与启动 | Function number 配置 | PCIe function 上下文 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| PF | PF0 BAR 中的 HDMA window，或本地 CPU 映射的同一寄存器空间 | 允许 | PF software / 本地 CPU | `PF=0, VF_EN=0` | PF0 Requester/Completer ID |
| VF | 项目自定义 VF BAR queue/doorbell/status window | 不允许 | PF software 或 SoC DMA virtualization logic 代理 | `PF=0, VF_EN=1, VF=0..3` | 对应 VF Requester/Completer ID |

VF doorbell 只用于向 PF software 或 SoC DMA virtualization logic 提交请求。实际 HDMA channel 通过写 `HDMA_DOORBELL_OFF_[WR|RD]CH_i.DB_START=1` 启动；channel 启动后，在停止前不得改写该 channel 的 context registers。

PF/VF FLR 不会自动停止与该 function 关联的 HDMA transfer。软件发起 FLR 前必须先禁止该 function 继续提交 DMA 请求，停止并回收其全部 HDMA channel，确认 transfer 已终止后再完成 FLR 流程。

### 4.4 AXI Outstanding Configuration

Synopsys AXI outstanding 相关资源按 AXI Manager 与 AXI Subordinate 分开配置。AXI Manager 侧使用 `CC_MAX_MSTR_TAGS_AXI` 配置 NP outstanding / ID 资源规模；AXI Subordinate 侧使用 `CC_MAX_SLV_TAG` 配置 outbound NP request outstanding 资源，并使用 `CC_SLV_NUM_OUTSTND_CPU_WR_REQ` 配置 NP write set-aside buffer。

<a id="tbl-axi-outstanding"></a>
**表 7 AXI Outstanding 配置**

| Profile | Controller | AXI Manager BDP 最低需求 | `CC_MAX_MSTR_TAGS_AXI` 配置上限 | `CC_MAX_SLV_TAG` | `CC_SLV_NUM_OUTSTND_CPU_WR_REQ` | 配置影响 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| X2 profile | C0 | 31 | 32 | 32 | 32 | BDP 最低需要 31 笔，`CC_MAX_MSTR_TAGS_AXI` 向上取合法档位 32；64 个 PCIe tags 中，32 个分配给 AXI Subordinate outbound NP request，32 个保留给 DMA read |
| X1 profile | C1 | 16 | 16 | 16 | 32 | AXI Manager NP outstanding / ID 资源按 X1 BDP 配置为 16；32 个 PCIe tags 中，16 个分配给 AXI Subordinate outbound NP request，16 个保留给 DMA read |

AXI Subordinate 接口支持通过 AXI bridge subordinate 发起 PCIe outbound NP read。该功能使用 `CC_MAX_SLV_TAG` 和非 HDMA PCIe tag pool。

AXI Subordinate 接口作为 CPU/软件通过寄存器发起非 DMA MMIO/config 访问的低吞吐路径，数据位宽配置为 64-bit，最大 burst length 配置为 8 beats，对应单 burst 最大 64B。该接口不支持 AXI WRAP burst，进入 PCIe Controller AXI Subordinate 地址窗口的上游 NoC/CPU 事务必须使用 INCR burst。

三个 AXI outstanding 相关参数的定义如下：

- `CC_MAX_MSTR_TAGS_AXI`：配置 AXI Manager 侧的 AXI master ID 数量和 AXI bridge 可跟踪的 Non-Posted outstanding 事务上限。该参数用于 PCIe inbound request 转换为本地 AXI Manager request 的路径，也就是 PCIe 侧事务进入 SoC/NoC 的方向。表中的 BDP 数字是满足目标吞吐所需的最低 outstanding 数，不是另一个硬件配置参数。`CC_MAX_MSTR_TAGS_AXI` 只能取 Reference 允许的离散档位，因此 X2 的最低需求 31 向上配置为 `32`，X1 的最低需求 16 直接配置为 `16`。该参数不是 PCIe tag pool，也不是 HDMA read tag reserve。
- `CC_MAX_SLV_TAG`：配置 AXI bridge slave 经 PCIe link 发出的 Non-Posted AXI request 在同一时刻允许 outstanding 的最大数量。该参数用于 CPU/SoC 通过 PCIe Controller AXI Subordinate 接口发起 PCIe outbound Non-Posted read/write 的路径。Reference 允许值为 2、4、8、16、32、64、128、256、512、768、1024、2048 和 3072，不允许配置任意整数。未启用 DMA 时默认值为 `CX_MAX_TAG+1`；启用 DMA 时默认值为 `(CX_MAX_TAG+1)/2`。本项目 X2/X1 均启用 DMA，因此分别配置为 `32`/`16`。
- `CC_SLV_NUM_OUTSTND_CPU_WR_REQ`：配置 AXI Subordinate 侧 Non-Posted write set-aside buffer 深度。该 buffer 用于在 NP read 被阻塞时暂存 AXI Subordinate 侧的 NP write，使 Posted write 可绕过被阻塞的 NP write，避免 AXI write channel 因 NP write 阻塞而影响 posted traffic 前进。Reference 合法值为 2、4、8、16、32、64，固定默认值为 32；最新 X2/X1 配置均未显式覆盖，因此两个 profile 均配置为 `32`。该参数不分配 PCIe tag，不增加 outbound NP read 能力，也不应与 `CC_NUM_DMA_RD_TAG` 作此消彼长的比较。

以上三个参数均属于 AXI bridge 资源配置。`CC_MAX_MSTR_TAGS_AXI` 作用于 AXI Manager 方向，`CC_MAX_SLV_TAG` 和 `CC_SLV_NUM_OUTSTND_CPU_WR_REQ` 作用于 AXI Subordinate 方向。`CX_MAX_TAG + 1` 定义 Controller outbound NP PCIe tag 总池，`CC_MAX_SLV_TAG` 定义其中供 AXI Subordinate outbound NP request 使用的份额，`CC_NUM_DMA_RD_TAG` 定义其中保留给 DMA read 的份额；必须满足 `CC_MAX_SLV_TAG + CC_NUM_DMA_RD_TAG <= CX_MAX_TAG + 1`。当前 X2 为 `32+32=64`，X1 为 `16+16=32`，均采用 1:1 划分。

### 4.5 AXI Link-Down Handling

本节定义 PCIe link down、hot reset 或 warm reset 期间 AXI Bridge 两个方向的 pending transaction 处理要求。Controller 使用 automatic flush 清理已经接受的 request，并在完成清理后进入 reset 流程。

#### AXI Subordinate Direction

AXI Subordinate read request 完成 AR handshake 并转换为 PCIe MRd 后，如果 PCIe link 在 Completion 返回前断开，Controller 必须通过 AXI Bridge automatic flush 结束 pending Non-Posted request，不得使 AXI transaction 永久处于 outstanding 状态。Link-down reset request 触发 flush 后，pending AXI Subordinate NP request 按 Completion Timeout 处理；已缓冲的完整 Completion 可正常返回，尚未完成的 read request 以 AXI error response 结束。Flush 完成后，Controller deassert `slv_*ready`，阻止新的 AXI Subordinate request 被接受。

本项目使用以下配置和软件约束：

- `LINK_FLUSH_CONTROL_OFF.AUTO_FLUSH_EN=1`，在 link down、hot reset 或 warm reset 时先清理 AXI Bridge pending requests，再执行 Controller/reset 流程。
- `AMBA_ERROR_RESPONSE_DEFAULT_OFF.AMBA_ERROR_RESPONSE_GLOBAL=1`，使 Non-Posted request 的 Completion error 映射为 AXI error，而不是 `OKAY` 加全 1 数据。
- `AMBA_ERROR_RESPONSE_DEFAULT_OFF.AMBA_ERROR_RESPONSE_MAP[5]=1`，将 Completion Timeout 映射为 `SLVERR`。AXI requester 收到非 `OKAY` response 后必须丢弃返回数据。
- PCIe Completion Timeout mechanism 必须保持启用。当 link 未明确进入 down 状态但远端 Completion 未返回时，由 Completion Timeout 最终结束 request。默认可编程 range 下 Controller timeout 约为 28ms~44ms；`ROUND_TRIP_LATENCY=1000ns` 只用于 Completion Queue sizing，不是运行时 Completion Timeout。

#### AXI Manager Direction

PCIe inbound MRd 转换为 AXI Manager read request，并在 AXI `AR` channel 完成 handshake 后，如果 PCIe link 在 SoC 返回 read response 前断开，Controller 进入 automatic flush。已经发往 SoC 的 AXI Manager request 继续执行；SoC 可以在 link down 后延迟返回 `RDATA/RRESP/RLAST`，Controller 必须继续接收完整 response 并将其丢弃，不再尝试通过已经断开的 PCIe link 返回 Completion。全部在途 AXI Manager transaction 完成清理后，Controller 才进入后续 reset 流程。

本项目不支持 SoC AXI subordinate 永久不返回 response 的场景。所有已完成 AXI address handshake 的 read/write request 必须在系统规定的有限时间内返回完整的 `R`/`B` response，包括 PCIe link down、hot reset 和 warm reset 期间。

#### Reset and Flush Constraints

系统 reset logic 必须遵循 Controller 的 graceful flush/reset 顺序，不得在 pending request 被终止前单独复位 AXI Bridge、AXI requester 或 `mstr_aresetn`，也不得在 NoC/target subordinate 中保留旧 request，以免产生 orphaned transaction、迟到 response 或 AXI ID 复用冲突。

### 4.6 PCIe Tag Configuration

HDMA read 使用 MRd TLP，占用 PCIe tag。HDMA write 使用 MWr TLP，不占用 PCIe tag。`CC_NUM_DMA_RD_TAG` 表示保留给 HDMA MRd 的 tag 数。`CX_MAX_TAG + 1` 表示 Controller outbound NP request 的总 tag pool。

<a id="tbl-pcie-tag"></a>
**表 8 PCIe Tag 配置**

| Profile | Controller | AXI Manager BDP Outstanding | HDMA Read Tag | Total PCIe Tag Pool | `CX_MAX_TAG` | `CX_REMOTE_MAX_TAG` | `CC_NUM_DMA_RD_TAG` | 配置影响 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| X2 profile | C0 | 31 | 32 | 64 | 63 | 31 | 32 | 32 tags 给 HDMA read，32 tags 给非 HDMA NP read |
| X1 profile | C1 | 16 | 16 | 32 | 31 | 31 | 16 | 16 tags 给 HDMA read，16 tags 给非 HDMA NP request |

该 tag 划分为两类请求预留独立资源：

- HDMA read 使用各 profile 的专用 tag reserve。X2 当前配置 32 tags；在 `MIN_RD_REQ_SIZE=128B`、RTT=1000ns 的最坏假设下不能覆盖 Gen5 x2 full-rate BDP，需通过实际 request size 和性能模型验证。
- AXI Subordinate outbound NP read 保留独立 tag pool，保证 CPU/软件非 DMA config/MMIO read 完成功能。

三个 PCIe tag 相关参数的定义如下：

- `CX_MAX_TAG`：配置 Controller outbound Non-Posted PCIe request 可使用的 tag pool 大小。Synopsys GUI 中显示的是 `CX_MAX_TAG + 1`，RTL 参数值表示最大 tag 编号，因为 PCIe tags 编号范围为 `0` 到 `CX_MAX_TAG`。该 tag pool 用于本端发出的 outbound MRd 等 Non-Posted request。HDMA read 和非 HDMA outbound NP request 共享该总 pool，并由 `CC_NUM_DMA_RD_TAG` 进行资源划分。
- `CC_NUM_DMA_RD_TAG`：配置从 `CX_MAX_TAG + 1` 总 PCIe tag pool 中保留给 HDMA MRd request TLP generation 的 tag 数量。该部分 tag 仅供 HDMA read 使用，不再分配给 AXI bridge subordinate 或 application non-HDMA outbound NP request。HDMA read channel 在 PCIe link 上同时 outstanding 的 MRd request 数量不超过 `CC_NUM_DMA_RD_TAG`。非 HDMA 可用 tag 数量为 `CX_MAX_TAG + 1 - CC_NUM_DMA_RD_TAG`。Reference Manual 允许值为 2、4、8、16、32、64、128、256、512。
- `CX_REMOTE_MAX_TAG`：配置 Controller forwarded Non-Posted request / Target Completion Lookup Table 的规模。该参数用于跟踪从 PCIe 接收并转发到 application/AXI 侧、等待本地 completion 返回的 Non-Posted request。它不控制 PCIe wire 侧能够接收和缓存多少 Non-Posted request；wire 侧接收能力由 PCIe flow-control credit 和相关 receive buffer 决定。X2/X1 均配置为 `31`，对应 32-entry forwarded NP request tracking 资源。

本项目 tag pool 划分如下：

- X2 profile：`CX_MAX_TAG=63`，total tag pool 为 64；`CC_NUM_DMA_RD_TAG=32`，因此 HDMA read 和非 HDMA outbound NP request 各使用 32 tags。按 `ROUND_TRIP_LATENCY=1000ns`、`MIN_RD_REQ_SIZE=128B` 和 Gen5 x2 单向有效带宽计算，1000ns BDP 约为 7877B，需要 `ceil(7877/128)=62` 个 outstanding read requests；当前 32-tag 配置只能覆盖 4096B 在途数据，128B/1us 场景的理论上限约为 4.096GB/s。若实际 HDMA MRd request 为 256B，32 tags 可覆盖 8192B，在相同 RTT 下可覆盖 Gen5 x2 BDP。
- X1 profile：`CX_MAX_TAG=31`，total tag pool 为 32；`CC_NUM_DMA_RD_TAG` 未显式覆盖，按 Reference 默认值配置为 16，因此 HDMA read 和非 HDMA outbound NP request 各使用 16 tags。在 `MIN_RD_REQ_SIZE=128B`、RTT=1000ns 下，16 tags 可覆盖 2048B 在途 read data，理论上限约 2.048GB/s，仍不能覆盖 Gen5 x1 BDP；实际 MRd request 为 256B 时可覆盖 4096B，理论上限约 4.096GB/s，可覆盖 Gen5 x1 约 3.939GB/s 的有效带宽。仍需通过实际 request size 和 RTT 验证。

`CX_MAX_TAG`、`CX_REMOTE_MAX_TAG` 和 `CC_NUM_DMA_RD_TAG` 属于 PCIe tag / request tracking 资源配置。`CX_MAX_TAG` 定义本端 outbound NP tag 总池；`CC_NUM_DMA_RD_TAG` 定义 HDMA read 在该总池中的保留份额；`CX_REMOTE_MAX_TAG` 定义 forwarded NP request 的本地 completion 跟踪规模。三者不等同于 AXI Manager outstanding，也不等同于 `CC_MAX_SLV_TAG`。AXI Subordinate outbound NP read 的实际可并发数量同时受 `CC_MAX_SLV_TAG` 和非 HDMA PCIe tag pool 限制。`MIN_RD_REQ_SIZE` 只用于 CoreConsultant 推荐 tag/Completion Queue 计算，不定义 AXI Manager 或 AXI Subordinate 单 request 大小。

### 4.7 Flow-Control Credit Configuration

PCIe flow-control credit 使用 Synopsys coreConsultant 自动计算结果作为配置基线。Databook 说明 coreConsultant 会根据 lane 数、Controller datapath width、`CX_MAX_MTU`、flow-control update latency、internal delay 和 PHY latency 自动计算 RX queue credit 与 buffer size；手动覆盖 `RADM_*_HCRD_VCn` 和 `RADM_*_DCRD_VCn` 时，coreConsultant 会同步调整对应 Header/Data RAM 深度。

本项目只使用 VC0。VC1~VC7 不启用。VC0 的 Posted、Non-Posted 和 Completion receive queue credit 配置策略如下：

<a id="tbl-credit-config"></a>
**表 9 Flow-Control Credit 配置策略**

| Credit 类型 | Synopsys 参数 | 配置策略 | 说明 |
| :--- | :--- | :--- | :--- |
| Posted Header/Data Credit | `RADM_PQ_HCRD_VC0` / `RADM_PQ_DCRD_VC0` | X2=`66/480`；X1=`33/240` | 用于接收 inbound MWr 等 Posted TLP；credit 与 posted receive queue buffer 绑定 |
| Non-Posted Header/Data Credit | `RADM_NPQ_HCRD_VC0` / `RADM_NPQ_DCRD_VC0` | X2=`66/66`；X1=`33/33` | 用于接收 inbound MRd/config/IO 等 NP TLP；NP data credit 主要覆盖带 payload 的 NP write，当前不是高吞吐主路径 |
| Completion Header/Data Credit | `RADM_CPLQ_HCRD_VC0` / `RADM_CPLQ_DCRD_VC0` | X2/X1=`0/0` | `0` 表示采用 infinite advertised Completion credit；实际接收能力由 Completion Queue management 和内部 queue depth 保证 |
| Completion Queue Overflow Protection | `RADM_SEL_CPLQ_OVFLW_PRVNTN_VC` | X2=`2`；X1=`2` | 两个 profile 均启用 Completion Queue Management；当可用 header/data 存储不足以接收未来 Completion 时，Core 节流本端 Non-Posted request |
| Completion Queue Management | `CX_CPLQ_MANAGEMENT_ENABLE` | X2/X1=`1` | 两个 profile 均启用 Completion Queue management；Completion Queue mode 为 Store-and-forward |
| Completion Queue Mode | `RADM_CPL_QMODE_VC0` | X2/X1 均配置为 `1` | Reference Manual 定义 `0x1` 为 Store-and-forward；该参数不选择 overflow-prevention mechanism |
| Flow-Control Scaling | `FC_SCALE_EN`、`RADM_*Q_HSCALE_VC0`、`RADM_*Q_DSCALE_VC0` | X2/X1：`1`、Header scale=`0x2`、Data scale=`0x1` | Header scale `0x2` 为 factor 4，Data scale `0x1` 为 factor 1；X2/X1 P/NP advertised Header credit 分别为 264/132 |

表 10 给出按 Gen5、MPS=256B、RTT=1000ns 计算的 credit 校核目标。该表用于对 coreConsultant 生成配置进行项目吞吐目标覆盖检查，不作为直接手填 `RADM_*_HCRD/DCRD` 的配置值。

<a id="tbl-credit-estimation"></a>
**表 10 RTT=1000ns Credit 校核目标**

| Profile | Lane | 1000ns BDP | Posted Header Credit | Posted Data Credit | NP Header Credit | NP Data Credit | Completion Header Credit | Completion Data Credit | 配置结论 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| X2 profile | x2 | 7877B | 32 | 493 | 32 | 66 | 0（infinite） | 493 | 配置值为 P/NP `66/480`、`66/66`；Completion 使用 infinite advertised credit 和 1907-entry data queue |
| X1 profile | x1 | 3939B | 16 | 247 | 16 | 33 | 0（infinite） | 247 | 配置值为 P/NP `33/240`、`33/33`；Completion 使用 infinite advertised credit 和 954-entry data queue |

Credit 计算口径如下：

- Posted credit、Non-Posted credit 和 Completion credit 是三组独立的 PCIe flow-control credit pool；每组分别包含 Header credit 和 Data credit。
- HDMA write 发出 MWr TLP，消耗 Posted Header credit 和 Posted Data credit，不消耗 PCIe tag。
- HDMA read 发出 MRd TLP，MRd request 不带 data payload，消耗 NP Header credit，不消耗 NP Data credit；HDMA read 的数据通过远端返回 CplD，CplD 消耗 Completion Header credit 和 Completion Data credit。
- NP Data credit 主要用于带 payload 的 Non-Posted write，例如 I/O write、Configuration write 或 AtomicOp。本项目不使用 NP write 作为高吞吐主路径，因此 NP Data credit 使用 coreConsultant 自动计算值。
- Completion Header credit 受 CplD 拆包方式影响，拆包与 RCB、MRRS/MPS、远端 completer 行为和 completion queue overflow prevention 策略相关。X2/X1 profile 均配置 `RADM_CPL_QMODE_VC0=1`，即 Completion Queue Store-and-forward mode；`RADM_SEL_CPLQ_OVFLW_PRVNTN_VC=2` 选择 Completion Queue Management，与 queue mode 是两个独立参数。该机制根据可用 Completion Queue header/data storage 节流本端发出的 Non-Posted requests，确保其未来返回的 Completion 有空间可存放。
- Posted Header credit 和 NP Header credit 的校核目标按 256B request/payload 粒度由 `ceil(BDP / 256B)` 计算，并按本项目 profile 配置取 32/16。
- Posted Data credit 和 Completion Data credit 的校核目标按 `ceil(BDP / 16B)` 计算。
- PCIe data credit 单位为 16B。
- Flit Mode 支持时，advertised header/data credit 必须满足 4-credit 对齐要求，由 coreConsultant 生成配置负责处理。

X2/X1 VC0 receive queue 的 Header scale field 均配置为 `0x2`（factor 4），Data scale field 均配置为 `0x1`（factor 1）。X2 的 `RADM_PQ_HCRD_VC0=66`、`RADM_NPQ_HCRD_VC0=66`，形成 264 个 Posted/Non-Posted advertised Header credits；X1 对应字段均为 33，形成 132 个 advertised Header credits。X2 的 Posted/Non-Posted Data credit 字段为 480/66，X1 为 240/33。VC1~VC7 不启用，不形成额外可用资源。

X2 Completion Queue calculator 配置为 `CPLQ_MNG_HDP=192`、`CPLQ_MNG_DDP=15248`，对应 `RADM_CPLQ_DDP_VC0=1907`；X1 配置为 `CPLQ_MNG_HDP=192`、`CPLQ_MNG_DDP=7624`，对应 `RADM_CPLQ_DDP_VC0=954`。这些字段分别描述 Completion Queue management 的 header/data storage 规划和 VC0 Completion data queue depth，单位与内部 RAM 组织不同，不能直接相互替代。X2/X1 的 `RADM_MAX_OUTSTD_P_REQ` 均配置为 64，用于限制 AXI Manager 路径可卸载并跟踪的 inbound Posted requests，不代表 PCIe advertised Posted Header credit。

### 4.8 Synopsys Define / Parameter Configuration

本节按参数在系统设计中的用途分类：先列基础架构与接口，再列影响吞吐和并发能力的性能/队列资源，随后列 Function、Interrupt、地址转换、RAS 和 BAR 等软件可见能力与资源，最后列集成默认值及报告选项。分类行仅用于组织阅读，不对应额外的 CoreConsultant 参数。

<a id="tbl-snps-params"></a>
**表 11 Synopsys Define / Parameter 配置**

| 功能 | Synopsys Define / Parameter | X2 Profile | X1 Profile | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| **A. 基础架构与接口** |  |  |  | Controller 类型、链路上限及 Controller/PHY/AMBA 接口基线 |
| Device Type | `CC_DEVICE_TYPE` | 2 | 2 | Databook 定义 2 为 Dual Mode，支持 pin-selectable EP/RP |
| AMBA interface | `AMBA_INTERFACE` | 3 | 3 | Databook 定义 3 为 AXI4 bridge |
| ACE-Lite enable | `CC_ACELITE_ENABLE` | 1 | 1 | 两个 profile 均启用 ACE-Lite；实际使用范围为 `[TBD]` |
| 最大 lane 数 | `CX_NL` | 2 | 1 | Controller 最大 link width |
| 最大 PCIe 速率 | `CX_MAX_PCIE_SPEED` | 5 | 5 | 最高支持 Gen5；Gen6 speed mode 不启用 |
| PIPE version | `CX_PIPE_VER` | 2 | 2 | Databook 定义 2 为 PIPE 4.4.1 |
| **B. 性能与队列资源** |  |  |  | HDMA、AXI outstanding、PCIe tag、Completion Queue 和 flow-control 资源 |
| Max Payload | `CX_MAX_MTU` | 256 | 256 | MPS=256B |
| HDMA enable | `CC_DMA_ENABLE` | 1 | 1 | 使能内部 DMA |
| Hyper DMA enable | `CX_DMA_PF_ENABLE` | 1 | 1 | 使用 HDMA |
| HDMA write channel | `CC_NUM_DMA_WR_CHAN` | 4 | 2 | 本地 application memory 到远端 link partner 的 channel 数 |
| HDMA read channel | `CC_NUM_DMA_RD_CHAN` | 4 | 2 | 远端 link partner 到本地 application memory 的 channel 数 |
| HDMA write LLQ depth | `CX_DMA_WR_LLQ` | 8 | 8 | 每个 write channel 的 linked-list descriptor prefetch queue depth；X2/X1 write overlay RAM 分别为 32/16 entries |
| HDMA read LLQ depth | `CX_DMA_RD_LLQ` | 8 | 8 | 每个 read channel 的 linked-list descriptor prefetch queue depth；X2/X1 read overlay RAM 分别为 32/16 entries |
| HDMA channel register spacing | `CX_DMAREG_CHADDR_SPACE` | 4 | 4 | 4 表示 `CHSEP_4K`，相邻 channel 配置寄存器窗口间隔 4KB |
| AXI Manager data width | `MASTER_BUS_DATA_WIDTH` | 256 | 256 | AXI Manager 数据位宽 256-bit |
| AXI Manager burst length | `CC_MSTR_BURST_LEN` | 32 | 32 | 256-bit 下 master MTU 为 1024B |
| AXI Manager clock frequency | `AXI_MSTR_CLK_FREQ` | 500 | 500 | AXI Manager clock frequency 为 500MHz |
| AXI Subordinate clock frequency | `AXI_SLV_CLK_FREQ` | 500 | 500 | AXI Subordinate clock frequency 为 500MHz |
| AXI dedicated DBI slave | `DBI_4SLAVE_POPULATED` | 0 | 0 | 不实例化 dedicated AXI DBI slave |
| AXI Manager NP outstanding / ID | `CC_MAX_MSTR_TAGS_AXI` | 32 | 16 | X2 配置 32；X1 按 16-request BDP 目标取最小充分值 16 |
| AXI Manager page boundary | `CC_MSTR_PAGE_BOUNDARY_PW` | 13 | 13 | AXI Manager burst page boundary 为 4KB |
| AXI Subordinate data width | `SLAVE_BUS_DATA_WIDTH` | 64 | 64 | AXI Subordinate 接口位宽 64-bit |
| AXI Subordinate ID width | `CC_SLV_BUS_ID_WIDTH` | 5 | 5 | X1 显式配置为 5；ID width 只定义 AXI ID 编码宽度，不增加 `CC_MAX_SLV_TAG` 可跟踪事务数 |
| AXI Subordinate burst length | `CC_SLV_BURST_LEN` | 8 | 8 | AXI Subordinate 最大 burst length 为 8 beats；64-bit 接口对应 64B |
| AXI Subordinate WRAP support | `CC_SLV_WRAP_ENABLE` | 0 | 0 | 不支持 AXI WRAP burst；上游 NoC/CPU 需约束为 INCR burst |
| AXI link-down automatic flush | `LINK_FLUSH_CONTROL_OFF.AUTO_FLUSH_EN` | 1 | 1 | Link down/hot reset/warm reset 时自动终止 pending request，防止 AXI outstanding 挂起 |
| AXI NP error response enable | `AMBA_ERROR_RESPONSE_DEFAULT_OFF.AMBA_ERROR_RESPONSE_GLOBAL` | 1 | 1 | Completion error 返回 AXI error；不得使用复位默认的 `OKAY` 加全 1 数据行为 |
| Completion Timeout response | `AMBA_ERROR_RESPONSE_DEFAULT_OFF.AMBA_ERROR_RESPONSE_MAP[5]` | 1 | 1 | 将 Completion Timeout 映射为 `SLVERR`；错误 read data 无效 |
| AXI Subordinate NP outstanding | `CC_MAX_SLV_TAG` | 32 | 16 | AXI bridge slave 经 PCIe link 发出的 NP AXI request 最大 outstanding 数；与 DMA read 各占总 PCIe tag pool 的一半 |
| AXI Subordinate NP write set-aside | `CC_SLV_NUM_OUTSTND_CPU_WR_REQ` | 32 | 32 | 两个 profile 均采用 Reference 默认深度；用于卸载受阻的 NP write，不分配 PCIe tag |
| DMA read tag reserve | `CC_NUM_DMA_RD_TAG` | 32 | 16 | 从 tag pool 中保留给 HDMA MRd |
| Outbound NP tag pool | `CX_MAX_TAG` | 63 | 31 | 实际 tag 数为参数值加 1；X2/X1 total pool 分别为 64/32 |
| Forwarded NP request LUT | `CX_REMOTE_MAX_TAG` | 31 | 31 | 两个 profile 均提供 32-entry forwarded NP request tracking 资源 |
| AXI Manager Posted outstanding | `RADM_MAX_OUTSTD_P_REQ` | 64 | 64 | 可卸载并跟踪的 inbound Posted request 数；不是 PCIe advertised Posted credit |
| Completion queue mode | `RADM_CPL_QMODE_VC0` | 1 | 1 | 两个 profile 均配置为 Store-and-forward |
| Completion queue overflow protection | `RADM_SEL_CPLQ_OVFLW_PRVNTN_VC` | 2 | 2 | mechanism 2：可用 CplQ header/data storage 不足时节流 Non-Posted request，防止未来 Completion 溢出 |
| Completion queue management | `CX_CPLQ_MANAGEMENT_ENABLE` | 1 | 1 | 两个 profile 均启用 Completion Queue management |
| Completion Queue sizing RTT | `ROUND_TRIP_LATENCY` | 1000 | 1000 | Completion Queue calculator 的预期 round-trip latency，单位 ns |
| Critical PCIe read request size | `MIN_RD_REQ_SIZE` | 128 | 128 | 希望维持吞吐的最小 read request size，用于推荐 tag 数计算 |
| Completion management header depth | `CPLQ_MNG_HDP` | 192 | 192 | Completion Queue calculator 的 header storage 规划值 |
| Completion management data depth | `CPLQ_MNG_DDP` | 15248 | 7624 | Completion Queue calculator 的 data storage 规划值；单位不可与 credit 直接等同 |
| Completion LUT byte count | `RADM_CPL_LUT_STORE_BYTE_CNT` | 1 | 1 | Completion LUT 存储 byte count 信息，用于 completion 检查和 debug |
| Application returned credit | `APP_RETURN_CRD_EN` | 0 | 0 | 不使用 application-returned Flow Control credit interface |
| VC0 Posted RX credit | `RADM_PQ_HCRD_VC0` / `RADM_PQ_DCRD_VC0` | 66 / 480 | 33 / 240 | Header scale factor 4 后，X2/X1 advertised Header credit 分别为 264/132；Data scale factor 均为 1 |
| VC0 Non-Posted RX credit | `RADM_NPQ_HCRD_VC0` / `RADM_NPQ_DCRD_VC0` | 66 / 66 | 33 / 33 | Header scale factor 4 后，X2/X1 advertised Header credit 分别为 264/132 |
| VC0 Completion RX data depth | `RADM_CPLQ_DDP_VC0` | 1907 | 954 | Completion data queue depth；与 `CPLQ_MNG_DDP` 的内部单位不同 |
| VC0 Completion advertised credit | `RADM_CPLQ_HCRD_VC0` / `RADM_CPLQ_DCRD_VC0` | 0 / 0 | 0 / 0 | `0` 表示 infinite advertised Completion Header/Data credit；内部 queue 由 management mechanism 保护 |
| Flow-control header scaling | `RADM_PQ_HSCALE_VC0` / `RADM_NPQ_HSCALE_VC0` / `RADM_CPLQ_HSCALE_VC0` | `0x2` / `0x2` / `0x2` | `0x2` / `0x2` / `0x2` | `0x2` 为 factor 4；仅 VC0 有效 |
| Flow-control data scaling | `FC_SCALE_EN` / `RADM_PQ_DSCALE_VC0` / `RADM_NPQ_DSCALE_VC0` / `RADM_CPLQ_DSCALE_VC0` | `1` / `0x1` / `0x1` / `0x1` | `1` / `0x1` / `0x1` / `0x1` | `0x1` 为 factor 1 |
| **C. Function 与协议能力** |  |  |  | Tag capability、SR-IOV Function 模型和 ARI 配置 |
| 10-bit tag | `CX_10BITS_TAG` | 1 | 1 | 启用 10-bit tag capability |
| 14-bit tag | `CX_14BITS_TAG` | 0 | 0 | 当前不使用 14-bit tag |
| SR-IOV enable | `CX_SRIOV_ENABLE` | 1 | 1 | 启用 SR-IOV |
| Physical Function 数量 | `CX_NFUNC` | 1 | 1 | 每个 Controller 在 EP mode 仅实现 PF0 |
| Internal VF 总数 | `CX_NVFUNC` | 4 | 4 | 均由 PF0 的 4 个 VF 派生 |
| PF0 VF 数量 | `CX_MAX_VF_0` | 4 | 4 | 每个 Controller 仅实现 PF0；PF1 相关参数不适用 |
| External VF | `CX_EXTENSIBLE_VFUNC` | 0 | 0 | 使用 internal VF |
| Dynamic VF allocation | `DYNAMIC_VF_ENABLE` | 0 | 0 | 使用 static VF allocation |
| VF stride always one | `CX_VF_STRIDE_ALWAYS_ONE` | 1 | 1 | VF Stride 固定为 1 |
| ARI enable | `CX_ARI_ENABLE` | 1 | 1 | SR-IOV 场景启用 ARI capability |
| **D. Interrupt 能力与资源** |  |  |  | PF/VF MSI、MSI-X capability、向量规模以及 Table/PBA 布局 |
| PF MSI capability | `MSI_CAP_ENABLE` | 1 | 1 | PF 支持 MSI |
| PF MSI application interface | `MSI_IO` | 1 | 1 | 启用 Synopsys MSI application interface |
| PF MSI per-vector mask | `MSI_PVM_EN` | 1 | 1 | 启用 MSI per-vector masking |
| PF MSI multi-message capable | `DEFAULT_MULTI_MSI_CAPABLE_0` | 5 | 5 | X2/X1 PF0 均最多支持 32 个 MSI messages |
| PF MSI 64-bit capable | `DEFAULT_MSI_64BIT_CAPABLE_0` | 1 | 1 | 支持 64-bit MSI message address |
| PF MSI PVM capable | `DEFAULT_MSI_PVM_CAPABLE_0` | 1 | 1 | MSI capability 暴露 per-vector mask/pending 支持 |
| Extended MSI data | `DEFAULT_EXT_MSI_DATA_CAPABLE` | 0 | 0 | 不启用 extended MSI data capability |
| PF MSI-X capability | `MSIX_CAP_ENABLE` | 1 | 1 | PF 支持 MSI-X |
| Integrated MSI-X | `MSIX_TABLE_EN` | 1 | 1 | PF MSI-X table/PBA 使用 Controller 内部实现 |
| MSI-X external I/O | `MSIX_IO` | 0 | 0 | 不使用外部 MSI-X table/PBA 和外部 MSI-X generation block |
| PF MSI-X table size | `MSIX_TABLE_SIZE_0` | `0x1f` | `0x1f` | PF0 配置 32 entries |
| PF MSI-X table offset | `MSIX_TABLE_OFFSET_0` | `0x40000` | `0x40000` | 参数编码为 bits `[31:3]`，对应 PF BAR0 byte offset `0x200000`；32-entry Table 占 `0x200000~0x2001ff` |
| PF MSI-X PBA location | `MSIX_PBA_BIR_0` / `MSIX_PBA_OFFSET_0` | BAR0 / derived | BAR0 / derived | Reference 默认基线紧随 Table：预期 byte offset `0x200200`，32 vectors 的 PBA 占 8B；以生成报告为最终值 |
| VF MSI capability | `VF_MSI_CAP_ENABLE` | 1 | 1 | VF 支持 MSI |
| VF MSI-X capability | `VF_MSIX_CAP_ENABLE` | 1 | 1 | 使用 SR-IOV/internal VF 条件下的 Reference 派生默认值，VF 支持 MSI-X |
| VF MSI-X table location | `VF_MSIX_TABLE_BIR_0` / `VF_MSIX_TABLE_OFFSET_0` | BAR0 / `0x1000` | BAR0 / `0x1000` | Table Offset field `0x1000` 对应 VF BAR0 byte offset `0x8000`；32-entry Table 占 `0x8000~0x81ff` |
| VF MSI-X PBA location | `VF_MSIX_PBA_BIR_0` / `VF_MSIX_PBA_OFFSET_0` | BAR0 / derived | BAR0 / derived | Reference 默认基线预期为 byte offset `0x8200`，32 vectors 的 PBA 占 8B；以生成报告为最终值 |
| **E. 功能支持与 RAS** |  |  |  | 可选协议能力、AER、PTM、ordering/coherency、Automotive 和错误报告 |
| TLP Prefix | N/A | 0 | 0 | X2/X1 profile 均不支持 TLP Prefix，不声明或处理 Local/End-to-End TLP Prefix capability |
| TPH requester | N/A | 0 | 0 | X2/X1 profile 均不支持 TPH Requester，不生成带 TPH Processing Hints 的请求 |
| FLR enable | `CX_FLR_ENABLE` | 1 | 1 | SR-IOV=1 时 FLR 支持由 SR-IOV 配置派生 |
| VF AER enable | `VF_AER_ENABLE` | 1 | 1 | 两个 profile 的 internal VF 均实现 AER Extended Capability |
| PTM enable | `CX_PTM_ENABLE` | 1 | 1 | 启用 Precision Time Measurement；role 相关 requester/responder 默认值及各速率 TX/RX latency 需在系统初始化时校准 |
| No Snoop supported | `DEFAULT_NO_SNOOP_SUPPORTED_0` | 1 | 1 | PF0 声明支持 No Snoop；必须与 ACE-Lite、NoC 和缓存一致性策略匹配 |
| ID-Based Ordering | `CX_IDO_ENABLE` | 1 | 1 | 启用 IDO；Controller 不增加额外 ordering，IDO ordering 由 application 保证 |
| Automotive safety package | `CX_AUTOMOTIVE_ENABLE` | 1 | 1 | 启用 PCIe Automotive safety package |
| Surprise Down reporting | `SURPRISE_LINK_DOWN_SUPPORTED` | 1 | 1 | 在 Downstream Port role 增加 Surprise Down error reporting protocol check |
| FC Watchdog default disable | `DEFAULT_FC_WATCH_DOG_DISABLE` | 1 | 1 | FC Watchdog Timer Disable bit 的复位默认值为 1 |
| **F. 地址转换与软件可见资源** |  |  |  | iATU region、AER log、Expansion ROM 以及 PF/VF BAR 类型和 sizing scheme |
| iATU enable | `CX_INTERNAL_ATU_ENABLE` | 1 | 1 | 使用 internal iATU |
| outbound iATU region | `CX_ATU_NUM_OUTBOUND_REGIONS` | 8 | 8 | OB region 数 |
| inbound iATU region | `CX_ATU_NUM_INBOUND_REGIONS` | 8 | 8 | IB region 数 |
| PF0 BAR register owner | `UNROLL_FUNC_NUM` / `UNROLL_BAR_NUM` | 0 / 0 | 0 / 0 | `UNROLL_FUNC_NUM=0`、`UNROLL_BAR_NUM=0`；EP mode 下，PL、unrolled iATU 和 unrolled HDMA 均映射至 PF0 BAR0，该 BAR 必须路由到 Target 0 |
| PF0 BAR PL register mapping | `ENABLE_MEM_MAP_PL_REG` / `PL_OFFSET_BAR` | 1 / `0x0` | 1 / `0x0` | 启用 PL register memory mapping；默认 `DISABLE_CFG_MAP_PL_REG=0` 时有效 byte range 固定为 `0x700~0x8ff` |
| PF0 BAR unrolled iATU mapping | `ENABLE_MEM_MAP_UNROLL_ATU_REG` / `UNROLL_ATU_OFFSET_BAR` | 1 / `0x10000` | 1 / `0x10000` | unrolled iATU register 从 PF0 BAR0 byte offset `0x10000` 开始 |
| PF0 BAR unrolled HDMA mapping | `ENABLE_MEM_MAP_UNROLL_DMA_REG` / `UNROLL_DMA_OFFSET_BAR` | 1 / `0x20000` | 1 / `0x20000` | unrolled HDMA register 从 PF0 BAR0 byte offset `0x20000` 开始；窗口大小由 channel 数和 `CX_DMAREG_CHADDR_SPACE` 派生 |
| AER header log depth | `CX_HDR_LOG_DEPTH_0` | `0x4` | `0x4` | PF0 配置 depth 4 |
| VF shared header log | `VF_HDR_LOG_SHARED` / `VF_HDR_LOG_SHARED_DEPTH` | 1 / 2 | 1 / 2 | PF0 的 4 个 VF 共用深度 2 的 Header Log；节省 RAM，但需验证并发错误记录需求 |
| Expansion ROM BAR | `ROM_BAR_ENABLED_0` | 0 | 0 | PF0 不包含 Expansion ROM BAR |
| PF BAR0 | `BAR0_TYPE_0` / `BAR0_SIZING_SCHEME_0` | 0 / 1 | 0 / 1 | 32-bit BAR，使用 Programmable Mask；因 MSI-X PBA 结束于预期 byte `0x200207`，aperture 至少需覆盖该地址，按 2 的幂次规划时不小于 4MiB |
| PF BAR1 | `BAR1_ENABLED_0` / `BAR1_SIZING_SCHEME_0` | 1 / 2 | 1 / 2 | 启用独立 BAR1，并使用 Resizable BAR sizing scheme |
| PF BAR2/3 | `BAR2_TYPE_0` / `BAR2_SIZING_SCHEME_0` | 2 / 2 | 2 / 2 | BAR2/3 组成 64-bit Resizable BAR |
| PF BAR4/5 | `BAR4_TYPE_0` / `BAR4_SIZING_SCHEME_0` | 2 / 2 | 2 / 2 | BAR4/5 组成 64-bit Resizable BAR |
| VF BAR0 | `VF_BAR0_TYPE_0` / `VF_MEM_FUNC0_BAR0_TARGET_MAP` | 0 / 0 | 0 / 0 | 32-bit VF BAR0，路由至 Target 0；需覆盖预期结束于 byte `0x8207` 的 MSI-X Table/PBA，按 2 的幂次规划时不小于 64KiB |
| VF BAR1 | `VF_BAR1_ENABLED_0` | 1 | 1 | 启用 VF BAR1；type、aperture 和 target map 需由生成报告及地址规划确认 |
| VF BAR2/3 | `VF_BAR2_TYPE_0` / `VF_PREFETCHABLE2_0` | 2 / 1 | 2 / 1 | 64-bit prefetchable VF BAR2/3；target map 由 SoC 地址规划统一配置 |
| VF BAR4/5 | `VF_BAR4_TYPE_0` / `VF_PREFETCHABLE4_0` | 2 / 1 | 2 / 1 | 64-bit prefetchable VF BAR4/5；target map 由 SoC 地址规划统一配置 |
| **G. 集成默认值与报告选项** |  |  |  | 只影响生成报告的选项、低功耗复位默认值及 RAM timing 约束 |
| Register report unroll view | `CX_UNROLL_VIEW` | N/A（DBI2 view） | N/A（DBI2 view） | 两个 profile 均选择 `CX_MEMORY_MAP_VIEW=2`；该报告选项与上面的 BAR memory-mapped unrolled register 功能无关 |
| Register report memory-map view | `CX_MEMORY_MAP_VIEW` | 2 | 2 | 选择生成报告中的 DBI2 memory-map view；仅影响 DocBook/HTML register report，不影响 RTL |
| L1 Substate power-on scale | `DEFAULT_L1SUB_PORT_T_POWER_ON_SCALE` | `0x0` | `0x0` | 两个 profile 均保留 L1 Substate T_POWER_ON 默认字段，但本项目 ASPM 仅支持 L0s/L1，L1 Substates 不支持 |
| L1 Substate power-on value | `DEFAULT_L1SUB_PORT_T_POWER_ON_VALUE` | 7 | 7 | 同上，L1 Substates 功能不启用 |
| PHY PERST on warm reset | `DEFAULT_PHY_PERST_ON_WARM_RESET` | 1 | 1 | warm reset 时默认触发 PHY PERST |
| Single-port RAM read access | `RAM1P_RD_ACCESS` | 600 | 600 | pclk=1GHz 下的 RAM timing constraint，单位 ps |
| Single-port RAM address setup | `RAM1P_ADDR_SU` | 250 | 250 | pclk=1GHz 下的 RAM timing constraint，单位 ps |
| Two-port RAM read access | `RAM2P_RD_ACCESS` | 600 | 600 | pclk=1GHz 下的 RAM timing constraint，单位 ps |
| Two-port RAM address setup | `RAM2P_ADDR_SU` | 250 | 250 | pclk=1GHz 下的 RAM timing constraint，单位 ps |

表中的 RAM timing `pclk=1GHz` 是 Controller RAM timing constraint 的基准时钟，不是 AXI Manager/Subordinate 的 `mstr_aclk` / `slv_aclk`。AXI 接口时钟统一为 500MHz。

EP mode 下，PF0 BAR0 的四组内部窗口按 byte address 规划为：PL `0x700~0x8ff`、unrolled iATU 从 `0x10000` 开始、unrolled HDMA 从 `0x20000` 开始、integrated PF MSI-X Table `0x200000~0x2001ff`，PBA 预期位于 `0x200200~0x200207`。`UNROLL_ATU_SIZE` 和 `UNROLL_DMA_SIZE` 由生成配置派生，必须以 CoreConsultant report 校验 iATU/HDMA 窗口结束地址和互不重叠。Databook 还要求承载这些 memory-mapped internal registers 的 PF0 BAR0 路由至 Target 0，且 Posted receive queue 不得配置为 bypass mode，否则该访问路径不能工作。

### 4.9 X2/X1 Profile Configuration Considerations

表 11 是 X2/X1 profile 的正式参数基线。本节将两个 profile 共用的补充参数分为 PIPE/PHY 集成与链路训练/EQ 两类，仅保留会影响 Controller-PHY 匹配、链路训练或低功耗行为的配置，不重复 PF/VF、AXI、iATU、Completion、MSI/MSI-X 和 AER 配置。最新 X1 CoreConsultant 配置确认本节所列 PIPE、PHY delay、RxStandby、Equalization、Receiver Margining 和链路训练参数与 X2 相同；lane 数及 tag/channel 等资源差异以表 11 为准。

<a id="tbl-x2-coreconsultant-analysis"></a>
**表 12 X2/X1 Profile 补充参数与集成考量**

| 配置组 | X2/X1 参数和值 | 配置考量 | 集成约束 |
| :--- | :--- | :--- | :--- |
| **A. PIPE / PHY 集成** |  | Controller 与外部 32G PHY 的接口、时序和低功耗编码 | Controller 和 PHY 两侧必须采用一致配置 |
| PCIe speed mode | Gen1/2：2<br>Gen3/4/5：4<br>Gen6：0 | 各 generation 的 MAC/PHY datapath mode；最高速率由表 11 定义 | Controller、PHY、PIPE width 和 core clock 的组合必须合法 |
| PIPE timing/handshake | `CX_PIPE_RETIMING_DEPTH=1`<br>`CX_PIPE43_ASYNC_HS_BYPASS=1` | 在表 11 已选定的 PIPE 4.4.1 接口上配置 retiming depth 和 asynchronous handshake bypass | Controller 与 PHY 的 PIPE timing/handshake 实现必须匹配 |
| PIPE powerdown encoding | `CX_PIPE43_PICPM_ENCODING=0x4`<br>`CX_PIPE43_P1_1_ENCODING=0x4`<br>`CX_PIPE43_P1_2_ENCODING=0x4`<br>`CX_PIPE43_P0_PICPM=1`<br>`CX_PIPE43_PICPM_P1=0` | 参数定义 PIPE 4.3/4.4 powerdown state 的编码及 P0/P1 与 P_ICPM 的映射 | Controller 与 PHY 必须使用相同编码 |
| P2.NoBeacon | `CX_P2NOBEACON_ENABLE=1` | Link 进入 L2 时，`mac_phy_powerdown` 输出 P2.NoBeacon encoding，而不是 P2 encoding | C0/C1 均启用 P2.NoBeacon |
| PHY selection | `PHY_TYPE=7` | 7 为 Custom PHY；PHY 位于 Controller 外部并通过标准 PIPE interface 连接 | C0/C1 按 Custom PHY 集成流程连接 Synopsys 32G PHY |
| PHY delay | `CX_PHY_TX_DELAY_PHY=16`<br>`CX_PHY_RX_DELAY_PHY=90` | PHY provider 给出以 PIPE clock cycle 表示的最坏 TX/RX delay；Controller 使用两者计算 ACK DLLP latency 和 Retry Buffer size | C0/C1 Retry Buffer 计算均采用 TX=16、RX=90 cycles |
| RxStandby | `CX_RXSTANDBY_CONTROL=0x46`<br>`CX_RXSTANDBY_DEFAULT=0` | Control bits 1/2/6 分别在 rate change、inactive lane 和 RxStandby handshake 条件下控制 `mac_phy_rxstandby`；default 参数定义 reset 后初值 | C0/C1 reset 后 RxStandby 默认值为 0，并启用上述三类控制条件 |
| No Equalization Needed | `CX_NO_EQ_NEEDED=0` | 控制是否支持 No Equalization Needed | C0/C1 均不支持 No Equalization Needed |
| **B. 链路训练、EQ 与能力通告** |  | LTSSM 训练默认值、Equalization、Receiver Margining 和软件可见 latency capability | 所通告能力不得超过 PHY 和系统实测能力 |
| L1 exit latency | `DEFAULT_L1_EXIT_LATENCY=0x6`<br>`DEFAULT_COMM_L1_EXIT_LATENCY=0x6` | 设置 conventional/common-clock Link Capabilities Register 的 L1 Exit Latency field；`0x6` 表示 32us~64us | C0/C1 两种 clock condition 均报告 32us~64us |
| Equalization | `CX_GEN3_EQ_COEF_CONV_SUPPORTED=1`<br>`CX_GEN3_EQ_COEFQ_DEPTH=8`<br>Gen3 PSET Req Vec=`0x370`<br>Gen4 PSET Req Vec=`0x370`<br>Gen5 PSET Req Vec=`0x270`<br>Gen3/4/5 FOM initial eval=1<br>Gen3/4/5 2ms eval disable=1<br>Gen3/4/5 feedback mode=1<br>Gen3/4/5 Phase 2/3 exit mode=1 | Coefficient queue 保存连续 evaluation 的远端 TX coefficient，用于判断收敛；depth 8 同时作用于 Gen3/Gen4。各速率 preset request vector 和 equalization 默认字段按列出的值实现 | PHY 与 Controller 的 coefficient encoding、preset 支持和退出条件必须一致 |
| Receiver Margining | Gen4/Gen5 timing steps=7<br>max timing offset=20<br>voltage steps=32<br>max voltage offset=5<br>voltage/timing sample rate=31<br>voltage supported=1<br>left-right timing indicator=1<br>up-down voltage indicator=1<br>error sampler indicator=1<br>sample reporting method=1<br>max lanes=15 | Gen4/Gen5 使用相同的 Lane Margining at the Receiver capability 和量化范围 | PHY 必须支持所声明的 timing/voltage 步数、offset、采样率、indicator 和 reporting method；C0/C1 实际分别使用 2/1 条 lane |
| Link training values | `CX_NFTS=200`<br>`DEFAULT_GEN2_N_FTS=200`<br>`CX_COMM_NFTS=200`<br>`DEFAULT_FAST_LINK_SCALING_FACTOR=3` | 设置 N_FTS 和 Fast Link Scaling Factor 相关字段 | C0/C1 均配置为 200 / 200 / 200 / 3 |

表 12 中的 PIPE powerdown encoding、PHY delay、RxStandby、Equalization 和 Receiver Margining 均依赖 Controller 与 PHY 两侧一致配置。相关验证任务统一列入第 6 节 Open Items，避免在参数章节重复维护。

---

## 5. Block Description

### 5.1 sys_pcie

`sys_pcie` 为 PCIe Controller 相关逻辑的上层集成 block，用于承载 PCIe Controller instance、Controller 侧配置/状态接口、AXI Manager/AXI Subordinate 接口、MSI/MSI-X/interrupt 相关接口、reset/clock 控制接口以及与 HSIO bus/PHY 侧的连接逻辑。

<a id="fig-sys-pcie"></a>**图 5 sys_pcie 结构图**

![图 5 sys_pcie 结构图](./images/sys_pcie_diagram.png)

`sys_pcie` 内部模块划分、寄存器窗口、中断汇聚、reset/clock 信号边界和与 `sys_hsio_hybrid` 的接口关系为 `[TBD]`。

### 5.2 sys_hsio_hybrid

`sys_hsio_hybrid` 使用 `PHY_HYBRID`，支持 PCIe X2 + 2 路 Ethernet 或 PCIe X1 + PCIe X1 + 2 路 Ethernet。

Lane 分配见 [表 2](#tbl-hybrid-lane)。

### 5.3 NOC / Bus

多个 PCIe Controller 和 10G Ethernet Controller 共享同一条 `hsio_bus` 上行通路。所有 Controller 的上行 DMA 通路接到 `hsio_bus` 同一个 slave-side 汇聚口，经过 `hsio_bus` 仲裁后，从同一个 master-side 出口进入 `hsio_sub_bus`。`hsio_sub_bus` 再根据地址将事务路由至 MNOC 或 SNOC。

PCIe Controller AXI Manager 接口规格为 `256bit @ 500MHz`，Ethernet manager 接口规格为 `128bit @ 500MHz`，`hsio_bus` 上行主干为 `256bit @ 500MHz`。由于多个 Controller 共享同一个上行汇聚口和同一个 master-side 出口，系统吞吐和延迟由 `hsio_bus` 仲裁、`hsio_sub_bus` 路由、MNOC/SNOC 目标带宽、NoC/DDR QoS 和背压共同决定。

NoC / Bus 需要覆盖以下并发场景：

- 所有 active PCIe Controller 和 Ethernet Controller 同时工作时，`hsio_bus` 上行仲裁需保证各 Controller 获得符合 QoS 配置的服务，避免任一 Controller 长时间无法获得上行带宽。
- RC mode 下 CPU/软件非 DMA config/MMIO 访问通过 CPU/SNOC 下行到 PCIe Controller AXI Subordinate，并进一步由 Controller 发起 PCIe config/MMIO TLP；该路径不经过 `hsio_bus` 上行 DMA 汇聚口。若同一 PCIe Controller 同时存在 HDMA 和 CPU/软件非 DMA config/MMIO 访问，需要在 Controller 内部仲裁、PCIe link 带宽、Non-Posted tag/credit 和 completion timeout 约束下保证后者可完成。
- `hsio_sub_bus` 向 MNOC AXI 256-bit 与向 SNOC AXI 64-bit 同时访问时的仲裁和背压。

### 5.4 APR

APR 用于 PCIe 和 Ethernet partial reset 控制，使各个 Controller 可独立复位，并在异常场景下实现故障隔离。APR 支持单 Controller 复位、局部链路恢复和异常状态清除，避免影响其他正常工作的 PCIe/Ethernet link。

### 5.5 CKM

CKM 用于监测 PHY PLL 输出时钟稳定性，并向复位/初始化流程提供时钟稳定状态。监测点、稳定判定条件、异常上报和复位联动策略为 `[TBD]`。

### 5.6 AHB Backdoor

AHB backdoor 通路用于访问 PHY 内部 SRAM，在 PHY 启动之前 load firmware。该路径属于 PHY bring-up 和 firmware load 相关后门访问路径。

### 5.7 SYS-ETH

系统包含两个 10G Ethernet Controller，位于 `sys_hsio_hybrid`。

<a id="tbl-eth-config"></a>
**表 13 Ethernet Controller 配置**

| Controller | PHY/Lane | AXI 位宽 | AXI Outstanding | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| ETH0 | `PHY_HYBRID` Lane2 | AXI 128-bit | 10 | 10G/5G Ethernet |
| ETH1 | `PHY_HYBRID` Lane3 | AXI 128-bit | 10 | 10G/5G Ethernet |

Ethernet manager 是 Ethernet 的 DMA 口，接口时钟为 500MHz，通过异步边界接入 `hsio_bus`，并与 PCIe 共享 NoC/DDR 资源。Ethernet AXI outstanding 基于 AXI 域 BDP 预算，单 request 有效数据为 128B，RTT 为 1000ns。

---

## 6. Open Items

- [ ] PHY 初始化与配置流程。
- [ ] Bifurcation 模式编码、寄存器配置及非法组合处理。
- [ ] `sys_pcie_eth_wrap` harden partition 边界、跨 boundary 信号、物理约束和 floorplan 约束。
- [ ] `sys_pcie` 内部模块划分、寄存器窗口、中断汇聚、reset/clock 信号边界和 HSIO 接口关系。
- [ ] 验证 Common Clock、SRNS 和 SRIS 模式的参考时钟分发、PHY elastic buffer、SKP 补偿和冷复位/热复位/partial reset 释放顺序；Gen5 clock rate difference 满足 SRNS 200ppm、SRIS 3200ppm，Gen4 及以下满足 SRNS 600ppm、SRIS 5600ppm。
- [ ] NoC/DDR QoS、仲裁策略、背压策略和 starvation 控制。
- [ ] IO 接口信号定义与中断映射。
- [ ] 低功耗策略，包括电源域、唤醒源、ASPM L0s/L1 进入/退出流程。
- [ ] 功能安全机制，包括 ECC、CRC、Data Protect、错误聚合和错误上报。
- [ ] Synopsys PCIe Gen5 Controller 的 HDMA descriptor、buffer depth、BAR、AER、ASPM、ECAM 等参数配置。
- [ ] VF BAR queue/doorbell/status window、HDMA interrupt 隔离和软件资源规划。
- [ ] 确认 NoC/CPU 访问 PCIe AXI Subordinate 地址窗口时只产生 INCR burst，且最大 burst length 不超过 8 beats。
- [ ] 验证 AXI Subordinate MRd 已发出但 Completion 返回前 link down 的场景：automatic flush 必须终止 pending request，AXI 返回 `SLVERR`，read data 被丢弃，AXI ID、PCIe tag 和 Completion LUT 资源全部释放，并在 flush 后停止接受新 request。
- [ ] 验证 PCIe Completion Timeout 保持启用，并检查 link-down、hot reset 和 warm reset 时 Controller、AXI Bridge、NoC requester 的 graceful flush/reset 顺序不存在 orphaned request。
- [ ] 验证 PCIe inbound MRd 已在 AXI Manager `AR` channel 完成 handshake、但 SoC 尚未返回 read response 时发生 link down：SoC 延迟返回的全部 `RDATA/RRESP/RLAST` 必须被 Controller 接收并丢弃，flush 完成后 AXI ID 和内部 outstanding 资源全部释放，PCIe link 上不得发送迟到 Completion。
- [ ] 验证 NoC 和所有 AXI target subordinate 对已接受 request 提供有界 `R`/`B` response；SoC 永久不返回 response 属于不支持场景，reset/flush 流程不得遗留 orphaned transaction 或迟到 response。
- [ ] 确认 `MSIX_TABLE_EN=1` 时 Synopsys integrated MSI-X module 暴露的 application trigger/doorbell 接口满足 PF MSI-X 使用需求。
- [ ] 从 X2/X1 CoreConsultant 生成报告确认 PF0 MSI-X PBA 的派生值为 BAR0 byte offset `0x200200`，并确认 PF0 BAR0 Programmable Mask 提供不少于 4MiB aperture。
- [ ] 验证 SoC 集成满足 `CC_ACELITE_ENABLE=1` 的接口和属性处理要求。
- [ ] 检查 X2 CoreConsultant 生成报告与单 PF 规格一致：`CX_NFUNC=1`、`CX_NVFUNC=4`，且不存在 PF1 capability、BAR 和中断资源。
- [ ] 检查 X1 CoreConsultant 生成报告与单 PF 规格一致：`CX_NFUNC=1`、`CX_MAX_VF_0=4`、`CX_NVFUNC=4`，且不存在 PF1 capability、BAR 和中断资源；X2/X1 PF MSI 均应为 `DEFAULT_MULTI_MSI_CAPABLE_0=5`。
- [ ] 验证 X2 `CC_NUM_DMA_RD_TAG=32` 在实际 HDMA MRd request size 和 RTT 下的吞吐；128B/1us 场景理论上限约 4.096GB/s，不能跑满 Gen5 x2。
- [ ] 验证 X1 `CX_MAX_TAG=31`、默认 `CC_NUM_DMA_RD_TAG=16` 在实际 HDMA MRd request size 和 RTT 下的吞吐；128B/1us 场景理论上限约 2.048GB/s，256B/1us 场景约 4.096GB/s，后者可覆盖 Gen5 x1 BDP。
- [ ] 验证 X2 的 4 个及 X1 的 2 个 write/read channel 在 `CX_DMA_WR_LLQ=8`、`CX_DMA_RD_LLQ=8` 下的 descriptor prefetch latency、overlay RAM 深度、QoS 和队列吞吐；软件初始化必须一次性为同方向全部 channel 冻结 `PF_DEPTH`。
- [ ] 将 X2/X1 `CX_DMAREG_CHADDR_SPACE=4` 对应的 4KB channel window 间隔落实到 SoC 地址表、寄存器头文件和软件驱动。
- [ ] 验证 X2/X1 profile 的 VF MSI-X Table/PBA 配置和软件隔离模型一致；确认派生 PBA byte offset 为 `0x8200`，两个 profile 的 VF BAR0 aperture 均不小于 64KiB。
- [ ] 用生成报告确认 X2/X1 的 `UNROLL_ATU_SIZE`、`UNROLL_DMA_SIZE` 及窗口结束地址；验证 PF0 BAR0 路由至 Target 0、Posted receive queue 非 bypass，并覆盖 PL、iATU、HDMA、MSI-X Table/PBA 的无重叠访问。
- [ ] 验证 X2/X1 Completion Queue 数值配置：HDP=`192/192`、DDP=`15248/7624`、VC0 Completion data queue depth=`1907/954`，P/NP Header/Data credit=`66/480,66/66` 与 `33/240,33/33`。
- [ ] 冻结 X2/X1 PF/VF BAR0~BAR5 的 aperture/mask、Resizable BAR 支持 size、target map 及软件用途；确认 64-bit BAR pair 和 prefetchable 属性符合系统地址规划。
- [ ] 验证 X2/X1 PTM 在 EP/RP 两种 role 下的 requester/responder、external master time 和 local clock granularity 配置；初始化时按 Gen1~Gen5 分别写入以 ns 表示的 TX/RX latency，并验证 PTM update 与 ASPM L0s/L1 的交互。
- [ ] 验证 PF0 的 4 个 VF 共用 `VF_HDR_LOG_SHARED_DEPTH=2` 时，多 VF 并发 AER 错误的记录、读取和清除行为满足诊断需求。
- [ ] 验证 `DEFAULT_NO_SNOOP_SUPPORTED_0=1` 与 `CC_ACELITE_ENABLE=1`、NoC 属性传播及系统缓存一致性策略匹配。
- [ ] 冻结 PF0 的 Vendor ID、Device ID、Revision ID 和 Subsystem ID；当前 `CX_DEVICE_ID_0=0` 不是最终产品标识。
- [ ] 验证 `CX_AUTOMOTIVE_ENABLE=1`、`CX_IDO_ENABLE=1` 和 `DEFAULT_FC_WATCH_DOG_DISABLE=1` 的系统集成约束。
- [ ] 核对 Controller 与 PHY 两侧的 PIPE powerdown encoding、PHY TX/RX delay、RxStandby、Equalization 和 Receiver Margining 配置一致。
- [ ] 从生成报告记录 replay timer adjustment 参数 `DEFAULT_REPLAY_ADJ` / `DEFAULT_GEN3_REPLAY_ADJ` 的最终数值并校核合法性。
- [ ] 确认 Gen5 x2/x1 下 PCIe native datapath width、PIPE width、core clock 与 PHY speed mode 的合法组合。

---

## 7. Change History

| 版本 | 日期 | 变更内容 |
| :--- | :--- | :--- |
| 0.1 | [TBD] | 初始 PHY 映射 |
| 0.2 | [TBD] | 修正 phy_hybrid lane 分配 |
| 0.3 | [TBD] | 增加 PCIe/ETH 详细特性 |
| 0.4 | [TBD] | 按公司模板重组章节结构 |
| 0.5 | [TBD] | 调整 §3/§4 结构：Functional Description 分 5 小节，Block Description 承接 PHY 内容 |
| 0.6 | [TBD] | 补充 §3.1 Architecture 顶层结构、组件互联、bifurcation 交换与数据路径 |
| 0.7 | [TBD] | 新增 Figure List / Table List 章节，为所有图表添加锚点与交叉引用 |
| 0.8 | [TBD] | 子系统更名为 sys_pcie_eth_pwr_wrap，新增 NOC 总线组件与 Block Description |
| 0.9 | [TBD] | 补充 §4.4 NOC 总线：hsio_bus 与 hsio_sub_bus 结构描述 |
| 1.0 | [TBD] | 重组 §3 结构：3.3 Design Description、3.4 Matrix、3.5 Sub-IPs、3.6 IO Interfaces、3.7 Interrupt、3.8 Address Mapping；NOC 内容从 §4.4 移至 §3.4 Matrix |
| 1.1 | 2026-05-17 | `sys_hsio_pcie` 改为 2×X2 + 1×X1；Lane3 由 bifurcation mux 在 X2 和 X1 之间静态切换；统一文档结构与交叉引用 |
| 1.2 | 2026-06-06 | `sys_hsio_pcie` 更新为 1×X4 + 1×X2 + 1×X1；X4 使用 Lane0~Lane3，X2 使用 Lane2/Lane3，X1 使用 Lane3 |
| 1.3 | 2026-07-04 | 按当前项目需求更新：`PHY_PCIE` 支持 C0 X4 或 C0 X2+C1 X2，C1 使用 Lane2/Lane3；`PHY_HYBRID` 支持 C2 X2 或 C2 X1+C3 X1，同时 Lane2/Lane3 连接 2 路 10G Ethernet；补充 boot-time dual mode、AXI M/S 和内部 DMA 需求 |
| 1.4 | 2026-07-15 | 按顶层架构信息更新：补充 `hsio_bus`/`hsio_sub_bus`、SNOC/MNOC 接入、AXI 256/128/64-bit 位宽、PCIe/ETH outstanding 配置以及公共控制模块描述 |
| 1.5 | 2026-07-15 | 统一 PHY 命名为 `PHY_PCIE` / `PHY_HYBRID`；3.4 仅保留 AXI 位宽；补充 AHB 后门访问 PHY SRAM、APR partial reset、CKM PLL 时钟稳定性监测；替换顶层架构占位图 |
| 1.6 | 2026-07-21 | 根据 Synopsys databook 梳理 PCIe Controller Feature List；补充 SR-IOV 术语；将未定配置统一标记为 [TBD]，并将文档措辞调整为规格定义口径 |
| 1.7 | 2026-07-21 | 明确 PCIe Controller 使用 HDMA；按 X4/X2/X1 profile 增加 HDMA channel、PF/VF、MSI-X entry 和 iATU region 差异化配置 |
| 1.8 | 2026-07-21 | 将差异化资源配置移入 PCIe-Core 章节；按 outstanding 选择 tag 数；按 RTT=1000ns 增加 credit 估算；补充 Synopsys define / parameter 配置表 |
| 1.9 | 2026-07-21 | 按 BDP 直接计算 AXI outstanding：PCIe 单 request 256B、Ethernet 单 request 128B、RTT=1000ns；同步更新 PCIe tag 与 Synopsys 参数配置 |
| 1.10 | 2026-07-21 | 区分 HDMA read tag 与 total PCIe tag pool；为非 HDMA NP request 保留 tag；澄清 MRd/MWr 对 tag 和 flow-control credit 的不同要求 |
| 1.13 | 2026-07-21 | 采用性能优先型 tag 划分：HDMA read tag 覆盖 Gen5 1us RTT BDP，同时为 AXI Subordinate outbound NP read 保留独立 tag pool |
| 1.14 | 2026-07-21 | 拆分 AXI Manager 与 AXI Subordinate outstanding 配置：新增 `CC_MAX_MSTR_TAGS_AXI`、`CC_MAX_SLV_TAG` 和 `CC_SLV_NUM_OUTSTND_CPU_WR_REQ` 参数表 |
| 1.15 | 2026-07-21 | 补充 SR-IOV / HDMA 软件模型：EP mode 启用 PF/VF，RC mode 不启用本端 VF；定义 VF queue/doorbell 经 PF software 或 SoC DMA virtualization logic 映射到 HDMA channel |
| 1.16 | 2026-07-21 | 从头整理文档结构：拆分 Feature、Functional Description、PCIe Controller Configuration 和 Block Description；统一术语、表格编号和规格口径 |
| 1.17 | 2026-07-21 | 修正 PHY refclk、PERST 和 lane reset 粒度：每个 PHY 1 路 refclk/PERST，`PHY_HYBRID` 接外部晶振，`PHY_PCIE` 使用 `repeat_clk`，lane reset 来自对应 Controller |
| 1.18 | 2026-07-21 | 删除 Feature List 中 SR-IOV 术语解释章节，SR-IOV 配置需求保留在 PCIe Controller Configuration 章节 |
| 1.19 | 2026-07-21 | 在 AXI Outstanding 配置章节补充 `CC_MAX_MSTR_TAGS_AXI`、`CC_MAX_SLV_TAG`、`CC_SLV_NUM_OUTSTND_CPU_WR_REQ` 的详细定义和适用路径 |
| 1.20 | 2026-07-22 | 修正表 7 中寄存器名含竖线导致的 Markdown 表格显示问题 |
| 1.21 | 2026-07-22 | 在 PCIe Tag 配置章节补充 `CX_MAX_TAG`、`CX_REMOTE_MAX_TAG`、`CC_NUM_DMA_RD_TAG` 的详细定义、资源划分和适用路径 |
| 1.22 | 2026-07-22 | 补充 NoC / Bus 拓扑描述：所有 Controller 上行 DMA 通路接入同一个 `hsio_bus` slave-side 汇聚口，并从同一个 master-side 出口通往 `hsio_sub_bus`/MNOC/SNOC |
| 1.23 | 2026-07-22 | 修正 NoC / Bus 并发场景描述：区分 HDMA/Ethernet 上行汇聚路径与 RC 侧 CPU/软件非 DMA config/MMIO 下行及 PCIe outbound 路径 |
| 1.24 | 2026-07-22 | 将 NoC / Bus 公平性场景调整为所有 active PCIe Controller 和 Ethernet Controller 同时工作，覆盖共享 `hsio_bus` 汇聚口的整体仲裁行为 |
| 1.25 | 2026-07-22 | 删除 NoC / Bus 并发场景中与全体 active Controller 并发场景重叠的 C0 X2 + C1 X2 局部组合 |
| 1.26 | 2026-07-22 | 按 databook 更新 flow-control credit 配置策略：VC0 credit 使用 coreConsultant 自动计算，Completion Queue 使用 management/overflow protection，并保留 RTT=1000ns 校核目标 |
| 1.27 | 2026-07-22 | 按图片占位文件名更新 Clock/Reset 图引用；新增 harden 切分章节和 `module_division.png`；新增 `sys_pcie` block 描述和 `sys_pcie_diagram.png` 占位图 |
| 1.28 | 2026-07-22 | 将表 11 中 CplD / Posted Data Credit 表述改为 Payload Data Credit 等效校核目标，并补充该列的计算口径和适用路径 |
| 1.29 | 2026-07-22 | 重构表 11，分列列出 Posted/Non-Posted/Completion 的 Header/Data credit 校核目标，并明确 HDMA read data 与 HDMA write data 使用不同 credit pool |
| 1.30 | 2026-07-22 | 明确不支持 TLP Prefix 及其相关 ATS/PASID/PRS/TPH 功能；澄清 SR-IOV 与 MSI/MSI-X capability 的关系 |
| 1.31 | 2026-07-22 | 将 PF/VF interrupt mode 调整为待定；保留 SR-IOV 启用时 MSI 或 MSI-X capability 至少启用一种的约束，不再将 MSI-X 作为已选定配置 |
| 1.32 | 2026-07-22 | 补充 `VF_MSI_CAP_ENABLE`，明确 VF MSI 与 VF MSI-X capability 具备独立配置开关；PF/VF interrupt mode 仍保持待定 |
| 1.33 | 2026-07-22 | 明确 PF 支持 MSI 和 MSI-X，VF 支持 MSI，VF 是否支持 MSI-X 保持待定 |
| 1.34 | 2026-07-22 | 简化 Feature List 中 ATS/PASID/PRS/TLP Prefix/TPH 表述，统一为不支持 |
| 1.35 | 2026-07-24 | 根据 notes 中 PCIe core 配置备忘补充 AXI Subordinate burst/wrap、PF MSI/MSI-X、Completion LUT、APP returned credit、AXI page boundary 和 RAM timing 参数 |
| 1.36 | 2026-07-27 | 按单 PHY 方案更新现行规格：移除 `sys_hsio_pcie` / `PHY_PCIE`，保留 `PHY_HYBRID` 上当时旧命名的 C2 X2、C3 X1 与两路 10G Ethernet，并删除 X4 Controller profile 配置 |
| 1.37 | 2026-07-27 | 根据 X2 CoreConsultant 配置补充当时旧命名 C2 的 X2 profile 配置分析：DM/Gen5/X2/PIPE4.4.1/AXI4/iATU/ARI/MSI/MSI-X/Completion Queue/AXI clock 等参数及待确认项 |
| 1.38 | 2026-07-29 | 按最新 X2 CoreConsultant 配置补充 2 PF、4/4 iATU、Completion calculator、PF0/PF1 MSI/MSI-X/AER、IDO、Automotive、Surprise Down、DBI/unroll 等参数；修正 `RADM_CPL_QMODE_VC0=1` 为 Store-and-forward，并按 128B/1000ns 重新校核 HDMA read tag |
| 1.39 | 2026-07-29 | 将全部 PCIe Controller AXI Manager/Subordinate 时钟统一冻结为 500MHz；删除与 1GHz 目标相关的待确认项，并明确 AXI outstanding 仍由 PCIe link BDP 决定 |
| 1.40 | 2026-07-29 | 按 X2 CoreConsultant 配置将 PF0 `MSIX_TABLE_SIZE_0` 更新为 `0x1f`（32 entries） |
| 1.41 | 2026-07-29 | 统一采用正式配置口径：清晰配置值视为已确定，未显式覆盖项采用 Reference/CoreConsultant 默认；补充 PF1/VF 默认 MSI-X、VF 数量、AER depth 和 Expansion ROM 配置及合理性检查 |
| 1.42 | 2026-07-29 | 按单 PF 产品需求将 X2/X1 profile 统一为 `CX_NFUNC=1`；X2 仅保留 PF0 及其 4 个 VF，`CX_NVFUNC` 修正为 4，并移除现行规格中的 PF1 配置与检查项 |
| 1.43 | 2026-07-29 | 冻结 X1 profile 为 1 PF、4 VF，`CX_NVFUNC=4`；X1 的 PF/VF MSI、MSI-X capability、向量规模及 Table/PBA 资源与 X2 profile 保持一致 |
| 1.44 | 2026-07-29 | 调整表 12 参数列排版，将组合参数逐项换行，降低第二列显示宽度 |
| 1.45 | 2026-07-29 | 重构 4.7/4.8：4.7 保留正式参数基线，4.8 仅保留 PIPE/PHY、链路训练、EQ、Receiver Margining 和低功耗补充参数；删除重复配置和重复待确认清单 |
| 1.46 | 2026-07-29 | 冻结 X2 tag 配置：`CC_NUM_DMA_RD_TAG=32`、`CX_MAX_TAG=63`，总池 64 tags，HDMA/non-HDMA 各 32；补充 128B/1us 场景的吞吐风险 |
| 1.47 | 2026-07-30 | 按当前双 Controller 架构重新连续编号：X2 Controller 由 C2 改为 C0，X1 Controller 由 C3 改为 C1 |
| 1.48 | 2026-07-31 | 按最新 X2 配置及 Databook/Reference 更新：HDMA 改为 4 写+4 读、LLQ=8、channel window=4KB，iATU 改为 8/8；补充 PTM、VF AER/shared Header Log、No Snoop、PF/VF BAR 和 EQ/Receiver Margining 配置；PF MSI-X Table offset 恢复为待地址规划冻结 |
| 1.49 | 2026-07-31 | 重构 4.7/4.8 参数表组织：按基础接口、性能与队列、Function、Interrupt、功能/RAS、地址资源、集成默认值分类，并将 X2 补充参数拆分为 PIPE/PHY 集成与链路训练/EQ 两组 |
| 1.50 | 2026-08-02 | 将 CPU 低吞吐访问路径统一表述为 CPU/软件通过寄存器发起的非 DMA MMIO/config 访问 |
| 1.51 | 2026-08-02 | 删除 3.2 节 Harden 切分图下方的待定说明段落 |
| 1.52 | 2026-08-02 | 明确 PCIe Separate Refclk 同时支持 SRNS 和 SRIS，并补充不同速率下的 clock rate difference 验证约束 |
| 1.53 | 2026-08-02 | 补充 AXI Subordinate outbound read 在 Completion 返回前 link down 时的处理：启用 automatic flush，将 Completion Timeout 映射为 `SLVERR`，并增加 outstanding 资源释放和 graceful reset 验证要求 |
| 1.54 | 2026-08-02 | 补充 AXI Manager link-down 处理：支持 SoC 在断链后延迟返回并由 Controller 接收、丢弃 response；明确不支持 SoC 永久不返回 AXI response |
| 1.55 | 2026-08-02 | 将 AXI Subordinate/Manager 的 link-down handling 从 outstanding 配置中移出并合并为独立 4.5 节，后续配置章节顺延编号 |
| 1.56 | 2026-08-02 | 按 Databook/Reference 复核 4.3：明确 HDMA 寄存器只能分配给 PF、Function Number 仅定义 PCIe function 上下文，并将 VF BAR queue/doorbell 标为项目自定义虚拟化接口；补充 channel 启动和 FLR 约束 |
| 1.57 | 2026-08-03 | 按最新 X1 CoreConsultant 配置更新：HDMA 为默认 2 写+2 读、LLQ=8、channel window=4KB，iATU 为 8/8；tag 总池改为 16 并按默认拆分为 HDMA/non-HDMA 各 8；补齐 AXI、MSI/MSI-X、AER/PTM/IDO/Automotive、BAR 及 PIPE/EQ 参数，并记录 X1 read 吞吐风险 |
| 1.58 | 2026-08-03 | 按最新 X2/X1 CoreConsultant 配置及 Databook/Reference 更新：X1 tag 总池改为 32 并默认拆分为 HDMA/non-HDMA 各 16，AXI Manager resource 改为 32；两 profile 的 PF MSI 统一为 32 messages，PF MSI-X Table 固定在 BAR0 byte `0x200000`；补充 PL/iATU/HDMA BAR0 memory map、X1 flow-control/Completion Queue 参数和 aperture 集成约束 |
| 1.59 | 2026-08-03 | 确认 TBU 的 ADB 接口连接至 TCU；明确 X1 `CC_MAX_MSTR_TAGS_AXI=32` 来自 AXI4 的 Reference 默认值而非截图显式覆盖，最终展开值由 CoreConsultant report 确认 |
| 1.60 | 2026-08-03 | 将 X1 `CC_MAX_MSTR_TAGS_AXI` 冻结为正式配置值 32；各参数表直接记录数字 32，不再作为待生成报告确认的默认派生项 |
| 1.61 | 2026-08-03 | 将正式参数表中的工具默认值占位全部数值化：补齐 X2/X1 `CX_REMOTE_MAX_TAG`、Posted tracking、Completion management/overflow、CplQ depth、VC0 P/NP/Cpl credit 及 Header/Data scale |
| 1.62 | 2026-08-03 | 将 `CC_MAX_SLV_TAG` 正式配置更新为 X2=64、X1=32，并明确实际 outbound NP wire-side 并发仍受 non-HDMA PCIe tag pool 32/16 限制 |
| 1.63 | 2026-08-03 | 明确 X2/X1 均不支持 TLP Prefix 和 TPH Requester，删除两项 CoreConsultant 参数名的 `[TBD]` 占位 |
| 1.64 | 2026-08-03 | 按 Reference 修正 `CC_MAX_SLV_TAG` 语义和数值：X2=32、X1=16；与 `CC_NUM_DMA_RD_TAG` 各占总 PCIe tag pool 的一半，并补充离散合法值与总量约束 |
| 1.65 | 2026-08-03 | 修正 `CC_SLV_NUM_OUTSTND_CPU_WR_REQ`：最新 X2/X1 均未覆盖该参数，统一采用 Reference 固定默认值 32；明确该 NP write set-aside buffer 与 DMA read tag reserve 相互独立 |
| 1.66 | 2026-08-03 | 按实际配置将 X2/X1 `RADM_SEL_CPLQ_OVFLW_PRVNTN_VC` 均更新为 2，并补充 Completion Queue Management 通过节流 Non-Posted request 防止未来 Completion 溢出的参数意义 |
| 1.67 | 2026-08-03 | 删除 Feature List 和 BAR 参数表中 aperture/size 的 `[TBD]` 占位，仅保留已确定的 Resizable BAR、BAR type、prefetchable 和 target-map 配置要求 |
| 1.68 | 2026-08-03 | 简化 TBU/ITS 连接描述：TBU ADB 连接 TCU，ITS AXI-Stream 连接 GIC |
| 1.69 | 2026-08-03 | 将 X1 `CC_MAX_MSTR_TAGS_AXI` 从 Reference 默认值 32 调整为 16，与 X1 的 16-request BDP 目标一致；X2 保持 32 |
| 1.70 | 2026-08-04 | 澄清 4.4 中 AXI Manager BDP 最低需求与 `CC_MAX_MSTR_TAGS_AXI` 配置上限：X2 为 31/32，X1 为 16/16，避免将计算目标与离散配置值混淆 |

---

**文档状态**：草稿，后续按 `[TBD]` 条目继续补充。
