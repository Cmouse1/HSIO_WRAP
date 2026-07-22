# 技术规格书：HDD HSIO PCIe/Ethernet Subsystem

**版本**：1.29
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
  - [4.5 PCIe Tag Configuration](#45-pcie-tag-configuration)
  - [4.6 Flow-Control Credit Configuration](#46-flow-control-credit-configuration)
  - [4.7 Synopsys Define / Parameter Configuration](#47-synopsys-define--parameter-configuration)
- [5. Block Description](#5-block-description)
  - [5.1 sys_pcie](#51-sys_pcie)
  - [5.2 sys_hsio_pcie](#52-sys_hsio_pcie)
  - [5.3 sys_hsio_hybrid](#53-sys_hsio_hybrid)
  - [5.4 NOC / Bus](#54-noc--bus)
  - [5.5 APR](#55-apr)
  - [5.6 CKM](#56-ckm)
  - [5.7 AHB Backdoor](#57-ahb-backdoor)
  - [5.8 SYS-ETH](#58-sys-eth)
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
- [表 2 sys_hsio_pcie Lane 分配](#tbl-pcie-lane)
- [表 3 sys_hsio_hybrid Lane 分配](#tbl-hybrid-lane)
- [表 4 AXI Matrix 位宽规划](#tbl-axi-matrix)
- [表 5 PCIe Controller Instance 配置](#tbl-pcie-controller-instance)
- [表 6 HDMA / SR-IOV 差异化资源配置](#tbl-hdma-sriov-profile)
- [表 7 SR-IOV / HDMA 软件模型](#tbl-sriov-hdma-sw-model)
- [表 8 AXI Outstanding 配置](#tbl-axi-outstanding)
- [表 9 PCIe Tag 配置](#tbl-pcie-tag)
- [表 10 Flow-Control Credit 配置策略](#tbl-credit-config)
- [表 11 RTT=1000ns Credit 校核目标](#tbl-credit-estimation)
- [表 12 Synopsys Define / Parameter 配置](#tbl-snps-params)
- [表 13 Ethernet Controller 配置](#tbl-eth-config)

---

## 1. Introduction

`sys_pcie_eth_pwr_wrap` 是 HSIO PCIe/Ethernet 多协议端口聚合子系统。子系统包含 4 个 PCIe Controller、2 个 10G Ethernet Controller、2 个 X4 32G-PHY，以及连接 SNOC/MNOC 的 HSIO 片上互联。

本规格定义当前项目对 Synopsys PCIe Gen5 Dual Mode Controller、PHY bifurcation、HDMA、SR-IOV、AXI 接口、Ethernet DMA 接口和公共控制模块的配置需求。未最终冻结的条目统一标记为 `[TBD]`。

---

## 2. Feature List

### 2.1 Subsystem Feature

- 子系统包含 4 个独立 PCIe Controller：1 个 max X4、2 个 max X2、1 个 max X1。
- 子系统包含 2 个 10G Ethernet Controller，固定使用 `PHY_HYBRID` Lane2/Lane3。
- 子系统包含 2 个 X4 32G-PHY：`PHY_PCIE` 与 `PHY_HYBRID`。
- PCIe 与 Ethernet Controller 共享 `hsio_bus`、`hsio_sub_bus` 以及 NoC/DDR 资源。
- PCIe Controller 与 PHY lane 通过启动期静态 bifurcation 配置连接，运行期间不进行 lane 动态重分配。
- PCIe Controller 支持 EP/RC dual mode，role 在启动阶段静态确定，运行期间不进行 EP/RC 动态切换。

### 2.2 PCIe Controller Feature

以下条目适用于全部 4 个 PCIe Controller。

- **Controller IP**：Synopsys PCIe Gen5 Dual Mode Controller。
- **PCIe 速率**：支持 Gen1 / Gen2 / Gen3 / Gen4 / Gen5，向下兼容，链路速率自动协商。
- **Link Width**：按 Controller instance 支持 X4、X2 或 X1。
- **Role**：支持 Endpoint 和 Root Complex/Root Port，启动阶段静态选择。
- **HDMA**：使用 Controller 内部 HDMA；HDMA channel 数按 X4/X2/X1 profile 差异化配置。
- **SR-IOV**：支持并启用；PF/VF 数量按 X4/X2/X1 profile 差异化配置。
- **MSI/MSI-X**：支持 MSI/MSI-X；SR-IOV 场景使用 MSI-X。
- **Legacy Interrupt**：支持。
- **ARI**：支持并启用，用于 SR-IOV function number 扩展。
- **iATU**：支持 internal iATU；region 数按 X4/X2/X1 profile 差异化配置。
- **FLR**：支持 Function Level Reset。
- **Partial Reset**：支持各 Controller 独立 partial reset。
- **Hot Reset**：支持软件控制延迟恢复机制。
- **AER**：支持 Advanced Error Reporting。
- **RAS DP**：支持 Data Protect 功能。
- **ASPM**：仅支持 L0s 和 L1；不支持 L1.1、L1.2、L2。
- **PCI-PM**：支持 PCI Power Management D-state；链路可进入 L1。
- **SRIS**：支持 Separate Refclk Independent SSC。
- **Virtual Channel**：支持 1 个 VC。
- **Max Payload Size**：配置为 256B。
- **PIPE 接口**：支持 PIPE 4.4.1。
- **AXI Manager 接口**：`256bit @ 1GHz`，作为 HDMA 主通道。
- **AXI Subordinate 接口**：`64bit @ 1GHz`，作为 CPU CSR/config/PIO 从通道，并支持 outbound NP read 完成功能。PIO 表示 Programmed I/O，即 CPU/软件通过寄存器访问方式发起的非 DMA MMIO/config 访问。
- **ATS/PASID/PRS**：不支持。
- **Hot-Plug**：不支持。
- **Resizable BAR**：支持，启用策略为 `[TBD]`。
- **PTM**：支持，启用策略为 `[TBD]`。
- **IDE / DPC / eDPC / NPEM / DPA / OBFF / Atomic Operation / TPH / TLP Prefix**：启用策略为 `[TBD]`。
- **功能安全**：支持，具体机制为 `[TBD]`。

### 2.3 PHY Bifurcation Feature

- `PHY_PCIE` 支持 X4 或 X2+X2 两种启动期 bifurcation 模式。
- `PHY_HYBRID` Lane0/Lane1 支持 PCIe X2 或 PCIe X1+X1；Lane2/Lane3 固定用于 10G Ethernet。
- PCIe 与 Ethernet 支持跨协议共享 `PHY_HYBRID`，PHY 内部使用不同 PLL。
- Lane reversal、lane numbering 和 PIPE port mapping 不受限制。
- 每个 PHY 具有 1 路 refclk。`PHY_HYBRID` 使用外部晶振输入 refclk；`PHY_PCIE` 使用 `PHY_HYBRID` 输出的 `repeat_clk`。
- 每个 PHY 具有 1 路 PERST。
- 每个 lane 具有独立 reset，lane reset 来自对应 Controller。
- 每个 bifurcated PCIe link 具有独立 LTSSM。
- Controller 支持小于最大 lane 数运行并释放未用 lane；释放 lane 可被其他 Controller 使用。

---

## 3. Functional Description

### 3.1 Architecture

`sys_pcie_eth_pwr_wrap` 采用 PHY-centric 架构，以 `PHY_PCIE` 与 `PHY_HYBRID` 两个 X4 32G-PHY 为核心，通过启动期静态 bifurcation 将 PHY lane 分配给 PCIe 和 Ethernet Controller。

<a id="fig-top-arch"></a>**图 1 顶层架构图**

![图 1 顶层架构图](./images/top_architecture.png)

子系统的主要组成如下：

- **PCIe Controller**：4 个独立 instance。C0/C1 位于 `sys_hsio_pcie`，C2/C3 位于 `sys_hsio_hybrid`。
- **Ethernet Controller**：2 个 10G Ethernet Controller，位于 `sys_hsio_hybrid`，固定使用 `PHY_HYBRID` Lane2/Lane3。
- **PHY**：`PHY_PCIE` 承载 PCIe-only lane；`PHY_HYBRID` 承载 PCIe + Ethernet 混合 lane。
- **Bus / NoC**：`hsio_bus` 汇聚 SNOC 下行访问、PCIe/Ethernet DMA 上行访问和 PHY SRAM 后门访问；`hsio_sub_bus` 将上行访问拆分至 MNOC 与 SNOC。
- **公共控制模块**：包含 `sys_ctrl`、CRG、`ecc_aggr`、CKM、RSC、XAF、APR 和 SRAM。

### 3.2 Harden Division

`sys_pcie_eth_wrap` 的 harden 切分以 `sys_pcie`、`sys_hsio_pcie`、`sys_hsio_hybrid`、HSIO bus/common control 以及 PHY/analog boundary 为主要边界。Harden 切分图用于定义各 harden partition 的模块归属、跨 harden 接口和后端集成边界。

<a id="fig-module-division"></a>**图 2 Harden 切分图**

![图 2 Harden 切分图](./images/module_division.png)

各 harden partition 的详细边界、跨 boundary 信号、物理约束和 floorplan 约束为 `[TBD]`。

### 3.3 Lane Assignment

<a id="tbl-controller-lane-summary"></a>
**表 1 Controller 与 Lane 资源汇总**

| Controller | 所属 Block | Max Lane | Active Mode | PHY/Lane | Role |
| :--- | :--- | :--- | :--- | :--- | :--- |
| C0 | `sys_hsio_pcie` | X4 | X4 或 X2 | `PHY_PCIE` Lane0~3 或 Lane0~1 | EP/RC boot-time dual mode |
| C1 | `sys_hsio_pcie` | X2 | X2 | `PHY_PCIE` Lane2~3 | EP/RC boot-time dual mode |
| C2 | `sys_hsio_hybrid` | X2 | X2 或 X1 | `PHY_HYBRID` Lane0~1 或 Lane0 | EP/RC boot-time dual mode |
| C3 | `sys_hsio_hybrid` | X1 | X1 | `PHY_HYBRID` Lane1 | EP/RC boot-time dual mode |
| ETH0 | `sys_hsio_hybrid` | X1 | X1 | `PHY_HYBRID` Lane2 | 10G/5G Ethernet |
| ETH1 | `sys_hsio_hybrid` | X1 | X1 | `PHY_HYBRID` Lane3 | 10G/5G Ethernet |

`PHY_PCIE` 支持以下组合：

- C0 X4 独占 `PHY_PCIE` Lane0~3。
- C0 X2 使用 Lane0/Lane1，同时 C1 X2 使用 Lane2/Lane3。

<a id="tbl-pcie-lane"></a>
**表 2 sys_hsio_pcie Lane 分配**

| Controller | Lane 分配 | AXI 位宽 | AXI Manager BDP Outstanding | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| C0 PCIe X4 Controller | Lane0~Lane3 或 Lane0~Lane1 | AXI 256-bit | 62 | X4 active mode 使用 `PHY_PCIE` 全部 Lane；X2 active mode 使用 Lane0/Lane1，并释放 Lane2/Lane3 |
| C1 PCIe X2 Controller | Lane2~Lane3 | AXI 256-bit | 31 | 使用 `PHY_PCIE` Lane2/Lane3；可与 C0 X2 active mode 同时使用 |

`PHY_HYBRID` 支持以下组合：

- C2 X2 使用 Lane0/Lane1，同时 ETH0/ETH1 使用 Lane2/Lane3。
- C2 X1 使用 Lane0，C3 X1 使用 Lane1，同时 ETH0/ETH1 使用 Lane2/Lane3。

<a id="tbl-hybrid-lane"></a>
**表 3 sys_hsio_hybrid Lane 分配**

| Controller | Lane 分配 | AXI 位宽 | AXI Manager BDP Outstanding | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| C2 PCIe X2 Controller | Lane0~Lane1 或 Lane0 | AXI 256-bit | 31 | X2 active mode 使用 Lane0/Lane1；X1 active mode 使用 Lane0 |
| C3 PCIe X1 Controller | Lane1 | AXI 256-bit | 16 | 在 X1+X1 PCIe 拆分模式下使用 |
| ETH0 | Lane2 | AXI 128-bit | 10 | 10G/5G Ethernet |
| ETH1 | Lane3 | AXI 128-bit | 10 | 10G/5G Ethernet |

### 3.4 Data Path

子系统包含以下主要事务路径：

- **SNOC 下行访问**：SNOC 事务通过 SNOC TNIU 进入 `hsio_bus`，访问 PCIe、Ethernet、PHY、公共控制寄存器和 SRAM 相关资源。
- **PCIe HDMA 上行访问**：PCIe Controller 通过 AXI Manager 发起 HDMA 事务，经 `hsio_bus` 进入 `hsio_sub_bus`，并根据地址路由到 MNOC 或 SNOC。
- **PCIe AXI Subordinate 访问**：CPU 通过 AXI Subordinate 访问 Controller CSR/config/PIO；该接口支持 outbound NP read 完成功能，不作为高吞吐数据通道。PIO 表示 CPU/软件发起的 programmed I/O 访问，属于非 DMA 访问路径。
- **Ethernet DMA 上行访问**：Ethernet manager 作为 Ethernet DMA 口，以 AXI 128-bit 接入 `hsio_bus`，并根据地址路由到 MNOC 或 SNOC。
- **PHY SRAM 后门访问**：AHB 通路访问 PHY 内部 SRAM，用于 PHY 启动前 load firmware。

### 3.5 Matrix

NOC 总线由 `hsio_bus` 和 `hsio_sub_bus` 组成。主要接口位宽规划如下。

<a id="tbl-axi-matrix"></a>
**表 4 AXI Matrix 位宽规划**

| 路径/接口 | 位宽 | 说明 |
| :--- | :--- | :--- |
| SNOC TNIU -> `hsio_bus` | AXI 64-bit | SNOC 下行访问 HSIO 子系统 |
| `hsio_bus` 主干 | AXI 256-bit | HSIO 内部主要数据通路 |
| `hsio_bus` -> `hsio_sub_bus` | AXI 256-bit | HSIO 上行进入 `hsio_sub_bus` |
| `hsio_sub_bus` -> MNOC INIU | AXI 256-bit | DDR 地址空间上行访问 |
| `hsio_sub_bus` -> SNOC INIU | AXI 64-bit | 非 DDR 地址空间上行访问 |
| PCIe Controller AXI Manager | AXI 256-bit | PCIe HDMA 主通道 |
| PCIe Controller AXI Subordinate | AXI 64-bit | CPU CSR/config/PIO 从通道 |
| Ethernet manager | AXI 128-bit | Ethernet DMA 通道 |

`hsio_bus` 包含以下通路：

- `snoc2hsio`：SNOC 到 HSIO 的下行通路，接口位宽为 AXI 64-bit。
- `hsio2ring`：HSIO 到 ring/sub-ring/`hsio_sub_bus` 的上行通路，主干位宽为 AXI 256-bit。所有 PCIe Controller 和 Ethernet Controller 的上行 DMA 通路接入 `hsio_bus` 同一个 slave-side 汇聚口，并从 `hsio_bus` 同一个 master-side 出口进入 `hsio_sub_bus`，再路由至 MNOC 或 SNOC。
- AHB 后门访问通路：访问 PHY 内部 SRAM，用于 PHY firmware load。
- 异步边界：PCIe、Ethernet、公共控制模块与 `hsio_bus` 之间存在多个 CDC 边界。

`hsio_sub_bus` 将 `hsio2ring` 上行数据拆分为以下两类：

- `hsio2mnoc`：路由 DDR 地址空间，接口位宽为 AXI 256-bit。
- `hsio2snoc`：路由非 DDR 地址空间，接口位宽为 AXI 64-bit。

ADB master/slave 与 TBU 的连接关系为 `[TBD]`。

### 3.6 Clock and Reset

<a id="fig-clock-structure"></a>**图 3 Clock 结构图**

![图 3 Clock 结构图](./images/clock_diagram.png)

<a id="fig-reset-structure"></a>**图 4 Reset 结构图**

![图 4 Reset 结构图](./images/reset_diagram.png)

时钟与复位结构定义如下：

- `PHY_HYBRID` 使用外部晶振输入 refclk。
- `PHY_PCIE` 使用 `PHY_HYBRID` 输出的 `repeat_clk` 作为 refclk。
- PERST 按 PHY 粒度配置，每个 PHY 具有 1 路 PERST。
- Lane reset 按 lane 粒度配置，每个 lane 接收来自对应 Controller 的独立 reset。
- PCIe LTSSM 按 Controller/link 独立运行。
- `PHY_HYBRID` 内 PCIe 与 Ethernet 使用 PHY 内部不同 PLL。
- CKM 监测 PHY PLL 输出时钟稳定性，并向复位/初始化流程提供时钟稳定状态。
- APR 提供 PCIe 和 Ethernet partial reset 控制，实现单 Controller 独立复位和异常隔离。
- Common Clock / SRIS 模式、参考时钟分发、冷复位/热复位/partial reset 释放顺序为 `[TBD]`。

### 3.7 Sub-IPs

子系统按功能划分为以下 Sub-IP：

- **PCIe-Core**：Synopsys PCIe Gen5 Dual Mode Controller，共 4 个独立 instance。
- **32g-PHY**：`PHY_PCIE` 与 `PHY_HYBRID` 两个 X4 PHY。
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

---

## 4. PCIe Controller Configuration

### 4.1 Controller Instance

PCIe-Core 采用 Synopsys PCIe Gen5 Controller，配置为 4 个独立 Controller instance。

<a id="tbl-pcie-controller-instance"></a>
**表 5 PCIe Controller Instance 配置**

| Controller | Profile | Max Lane | Active Mode | PHY/Lane | AXI Manager | AXI Subordinate | AXI Manager BDP Outstanding | `CC_MAX_MSTR_TAGS_AXI` |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| C0 | X4 profile | X4 | X4 或 X2 | `PHY_PCIE` Lane0~3 或 Lane0~1 | AXI 256-bit @ 1GHz | AXI 64-bit @ 1GHz | 62 | 64 |
| C1 | X2 profile | X2 | X2 | `PHY_PCIE` Lane2~3 | AXI 256-bit @ 1GHz | AXI 64-bit @ 1GHz | 31 | 32 |
| C2 | X2 profile | X2 | X2 或 X1 | `PHY_HYBRID` Lane0~1 或 Lane0 | AXI 256-bit @ 1GHz | AXI 64-bit @ 1GHz | 31 | 32 |
| C3 | X1 profile | X1 | X1 | `PHY_HYBRID` Lane1 | AXI 256-bit @ 1GHz | AXI 64-bit @ 1GHz | 16 | 16 |

AXI Manager BDP outstanding 基于 AXI 域 BDP 计算，单 request 有效数据为 256B，RTT 为 1000ns。该值用于规划本地 NoC/DDR 侧 HDMA request 并发能力，不等同于 PCIe tag 数量。

### 4.2 HDMA / SR-IOV Resource Profile

<a id="pcie-core-hdma-sriov-profile"></a>
<a id="tbl-hdma-sriov-profile"></a>
**表 6 HDMA / SR-IOV 差异化资源配置**

| Profile | Controller | HDMA Channel | PF/VF 配置 | MSI-X 配置 | iATU Region 配置 | 配置影响 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| X4 profile | C0 | 4 channels：2 H2D + 2 D2H | 1 PF + 8 VF | PF 16 entries；每 VF 4 entries | OB 8；IB 8 | 覆盖 X4 Gen5 高吞吐场景；资源、验证和时序压力最高 |
| X2 profile | C1、C2 | 2 channels：1 H2D + 1 D2H | 1 PF + 4 VF | PF 8 entries；每 VF 2 entries | OB 4；IB 4 | 覆盖 X2 Gen5 中等吞吐场景；资源规模低于 X4 profile |
| X1 profile | C3 | 2 channels：1 H2D + 1 D2H | 1 PF + 2 VF | PF 4 entries；每 VF 2 entries | OB 2；IB 2 | 覆盖 X1 Gen5 基础吞吐场景；资源占用最小 |

Controller 在低于最大 lane 数运行时仍使用所属 profile，不进行运行期资源裁剪。C0 在 X2 active mode 下仍使用 X4 profile。

### 4.3 SR-IOV / HDMA Software Model

SR-IOV 作为 EP mode 功能启用。RC mode 下本端不启用 VF；RC mode 用于枚举、配置和访问下游 SR-IOV EP 的 PF/VF。

每个 Controller 配置 1 个 PF。VF 数量按 X4/X2/X1 profile 分别配置为 8/4/2。SR-IOV 使用 internal VF，不使用 external VF；VF allocation 使用 static allocation，不支持 VF migration。VF routing ID 初始配置为 `First VF Offset = 1`、`VF Stride = 1`。X4 profile 的 1 PF + 8 VF 依赖 ARI-capable hierarchy。

HDMA 寄存器归属 PF。VF 不直接访问 HDMA 全局配置寄存器。VF 通过 VF BAR 中的 queue/doorbell/status window 提交 DMA 请求；PF software 或 SoC DMA virtualization logic 将 VF 请求映射到 HDMA channel，并配置 `HDMA_FUNC_NUM_OFF_[WR|RD]CH_i` 中的 PF/VF/VF_EN 字段。

<a id="tbl-sriov-hdma-sw-model"></a>
**表 7 SR-IOV / HDMA 软件模型**

| 发起方 | 软件可见入口 | HDMA 寄存器配置方 | Function number 配置 | PCIe Requester/Completer ID |
| :--- | :--- | :--- | :--- | :--- |
| PF | PF BAR 或本地 CPU CSR | PF software / 本地 CPU | `HDMA_FUNC_NUM_OFF_WRCH_i` / `HDMA_FUNC_NUM_OFF_RDCH_i`：`PF=0, VF_EN=0` | PF requester/completer ID |
| VF | VF BAR queue/doorbell/status window | PF software 或 SoC DMA virtualization logic | `HDMA_FUNC_NUM_OFF_WRCH_i` / `HDMA_FUNC_NUM_OFF_RDCH_i`：`PF=0, VF_EN=1, VF=<VF number>` | VF requester/completer ID |

VF doorbell 用于 VF 向 PF software 或 SoC DMA virtualization logic 提交 DMA 请求。HDMA doorbell 用于启动实际 HDMA channel。实际 HDMA 启动通过写 `HDMA_DOORBELL_OFF_[WR|RD]CH_i.DB_START = 1` 完成。

### 4.4 AXI Outstanding Configuration

Synopsys AXI outstanding 相关资源按 AXI Manager 与 AXI Subordinate 分开配置。AXI Manager 侧使用 `CC_MAX_MSTR_TAGS_AXI` 配置 NP outstanding / ID 资源规模；AXI Subordinate 侧使用 `CC_MAX_SLV_TAG` 配置 outbound NP request outstanding 资源，并使用 `CC_SLV_NUM_OUTSTND_CPU_WR_REQ` 配置 NP write set-aside buffer。

<a id="tbl-axi-outstanding"></a>
**表 8 AXI Outstanding 配置**

| Profile | Controller | AXI Manager BDP Outstanding | `CC_MAX_MSTR_TAGS_AXI` | `CC_MAX_SLV_TAG` | `CC_SLV_NUM_OUTSTND_CPU_WR_REQ` | 配置影响 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| X4 profile | C0 | 62 | 64 | 64 | 32 | AXI Manager 覆盖 X4 本地 AXI BDP；AXI Subordinate 保留 64 个 outbound NP request tags |
| X2 profile | C1、C2 | 31 | 32 | 32 | 16 | AXI Manager 覆盖 X2 本地 AXI BDP；AXI Subordinate 保留 32 个 outbound NP request tags |
| X1 profile | C3 | 16 | 16 | 16 | 8 | AXI Manager 覆盖 X1 本地 AXI BDP；AXI Subordinate 保留 16 个 outbound NP request tags |

AXI Subordinate 接口支持通过 AXI bridge subordinate 发起 PCIe outbound NP read。该功能使用 `CC_MAX_SLV_TAG` 和非 HDMA PCIe tag pool。

三个 AXI outstanding 相关参数的定义如下：

- `CC_MAX_MSTR_TAGS_AXI`：配置 AXI Manager 侧的 AXI master ID 数量和 AXI bridge 可跟踪的 Non-Posted outstanding 事务资源。该参数用于 PCIe inbound request 转换为本地 AXI Manager request 的路径，也就是 PCIe 侧事务进入 SoC/NoC 的方向。Synopsys AXI bridge 在该路径上为 Non-Posted read request 和 Non-Posted write request 分配 AXI ID，并使用该参数配置 manager completion RAM、ID bus 宽度和相关 outstanding 跟踪资源。本项目按 AXI Manager BDP outstanding 向上取整配置该参数，X4/X2/X1 profile 分别配置为 64/32/16。该参数不是 PCIe tag pool，也不是 HDMA read tag reserve。
- `CC_MAX_SLV_TAG`：配置 AXI Subordinate 侧 outbound Non-Posted request 的 internal application tag 数量。该参数用于 CPU/SoC 通过 PCIe Controller AXI Subordinate 接口发起 PCIe outbound Non-Posted read/write 的路径，也就是 AXI Subordinate request 被 bridge 转换为 PCIe Non-Posted TLP 的方向。AXI Subordinate outbound request 需要同时占用 internal application tag 和 PCIe tag；当 internal application tag 或 PCIe tag 被耗尽时，AXI bridge subordinate interface 对 AXI fabric 施加 back-pressure。本项目将 `CC_MAX_SLV_TAG` 配置为与非 HDMA PCIe tag pool 对齐，X4/X2/X1 profile 分别配置为 64/32/16，用于保证 AXI Subordinate outbound NP read 完成功能。
- `CC_SLV_NUM_OUTSTND_CPU_WR_REQ`：配置 AXI Subordinate 侧 Non-Posted write set-aside buffer 深度。该 buffer 用于在 NP read 被阻塞时暂存 AXI Subordinate 侧的 NP write，使 Posted write 可绕过被阻塞的 NP write，避免 AXI write channel 因 NP write 阻塞而影响 posted traffic 前进。本项目按 `CC_MAX_SLV_TAG` 的一半配置该参数，X4/X2/X1 profile 分别配置为 32/16/8。该参数不增加 outbound NP read 能力，也不增加 PCIe tag pool。

以上三个参数均属于 AXI bridge 资源配置。`CC_MAX_MSTR_TAGS_AXI` 作用于 AXI Manager 方向，`CC_MAX_SLV_TAG` 和 `CC_SLV_NUM_OUTSTND_CPU_WR_REQ` 作用于 AXI Subordinate 方向；三者与 `CX_MAX_TAG + 1`、`CC_NUM_DMA_RD_TAG` 属于不同层级的资源。`CX_MAX_TAG + 1` 定义 Controller outbound NP PCIe tag pool，`CC_NUM_DMA_RD_TAG` 从该 pool 中保留 HDMA read tags，剩余 tag pool 供 AXI Subordinate outbound NP read/write 使用。

### 4.5 PCIe Tag Configuration

HDMA read 使用 MRd TLP，占用 PCIe tag。HDMA write 使用 MWr TLP，不占用 PCIe tag。`CC_NUM_DMA_RD_TAG` 表示保留给 HDMA MRd 的 tag 数。`CX_MAX_TAG + 1` 表示 Controller outbound NP request 的总 tag pool。

<a id="tbl-pcie-tag"></a>
**表 9 PCIe Tag 配置**

| Profile | Controller | AXI Manager BDP Outstanding | HDMA Read Tag | Total PCIe Tag Pool | `CX_MAX_TAG` | `CX_REMOTE_MAX_TAG` | `CC_NUM_DMA_RD_TAG` | 配置影响 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| X4 profile | C0 | 62 | 64 | 128 | 127 | 127 | 64 | 64 tags 给 HDMA read，64 tags 给非 HDMA NP read |
| X2 profile | C1、C2 | 31 | 32 | 64 | 63 | 63 | 32 | 32 tags 给 HDMA read，32 tags 给非 HDMA NP read |
| X1 profile | C3 | 16 | 16 | 32 | 31 | 31 | 16 | 16 tags 给 HDMA read，16 tags 给非 HDMA NP read |

该 tag 划分同时满足两类需求：

- HDMA read tag 数覆盖 RTT=1000ns、256B request 下的 Gen5 full-rate BDP 需求。
- AXI Subordinate outbound NP read 保留独立 tag pool，保证 CPU PIO/config/memory read 完成功能。

三个 PCIe tag 相关参数的定义如下：

- `CX_MAX_TAG`：配置 Controller outbound Non-Posted PCIe request 可使用的 tag pool 大小。Synopsys GUI 中显示的是 `CX_MAX_TAG + 1`，RTL 参数值表示最大 tag 编号，因为 PCIe tags 编号范围为 `0` 到 `CX_MAX_TAG`。因此表 9 中 X4/X2/X1 profile 的 `CX_MAX_TAG=127/63/31` 分别表示 total PCIe tag pool 为 128/64/32。该 tag pool 用于本端发出的 outbound MRd 等 Non-Posted request。HDMA read 和非 HDMA outbound NP request 共享该总 pool，并由 `CC_NUM_DMA_RD_TAG` 进行资源划分。
- `CC_NUM_DMA_RD_TAG`：配置从 `CX_MAX_TAG + 1` 总 PCIe tag pool 中保留给 HDMA MRd request TLP generation 的 tag 数量。该部分 tag 仅供 HDMA read 使用，不再分配给 AXI bridge subordinate 或 application non-HDMA outbound NP request。HDMA read channel 在 PCIe link 上同时 outstanding 的 MRd request 数量不超过 `CC_NUM_DMA_RD_TAG`。非 HDMA 可用 tag 数量为 `CX_MAX_TAG + 1 - CC_NUM_DMA_RD_TAG`。本项目 X4/X2/X1 profile 分别配置 `CC_NUM_DMA_RD_TAG=64/32/16`，对应非 HDMA tag reserve 为 64/32/16。
- `CX_REMOTE_MAX_TAG`：配置 Controller forwarded Non-Posted request / Target Completion Lookup Table 的规模。该参数用于跟踪从 PCIe 接收并转发到 application/AXI 侧、等待本地 completion 返回的 Non-Posted request。它不控制 PCIe wire 侧能够接收和缓存多少 Non-Posted request；wire 侧接收能力由 PCIe flow-control credit 和相关 receive buffer 决定。本项目将 `CX_REMOTE_MAX_TAG` 与 `CX_MAX_TAG` 对齐，X4/X2/X1 profile 分别配置为 127/63/31，使 forwarded NP request 跟踪资源与本 profile 的 tag 资源规模一致。

本项目 tag pool 划分如下：

- X4 profile：`CX_MAX_TAG=127`，total tag pool 为 128；其中 HDMA read 使用 64 tags，非 HDMA outbound NP request 保留 64 tags。
- X2 profile：`CX_MAX_TAG=63`，total tag pool 为 64；其中 HDMA read 使用 32 tags，非 HDMA outbound NP request 保留 32 tags。
- X1 profile：`CX_MAX_TAG=31`，total tag pool 为 32；其中 HDMA read 使用 16 tags，非 HDMA outbound NP request 保留 16 tags。

`CX_MAX_TAG`、`CX_REMOTE_MAX_TAG` 和 `CC_NUM_DMA_RD_TAG` 属于 PCIe tag / request tracking 资源配置。`CX_MAX_TAG` 定义本端 outbound NP tag 总池；`CC_NUM_DMA_RD_TAG` 定义 HDMA read 在该总池中的保留份额；`CX_REMOTE_MAX_TAG` 定义 forwarded NP request 的本地 completion 跟踪规模。三者不等同于 AXI Manager outstanding，也不等同于 `CC_MAX_SLV_TAG`。AXI Subordinate outbound NP read 的实际可并发数量同时受 `CC_MAX_SLV_TAG` 和非 HDMA PCIe tag pool 限制。

### 4.6 Flow-Control Credit Configuration

PCIe flow-control credit 使用 Synopsys coreConsultant 自动计算结果作为配置基线。Databook 说明 coreConsultant 会根据 lane 数、Controller datapath width、`CX_MAX_MTU`、flow-control update latency、internal delay 和 PHY latency 自动计算 RX queue credit 与 buffer size；手动覆盖 `RADM_*_HCRD_VCn` 和 `RADM_*_DCRD_VCn` 时，coreConsultant 会同步调整对应 Header/Data RAM 深度。

本项目只使用 VC0。VC1~VC7 不启用。VC0 的 Posted、Non-Posted 和 Completion receive queue credit 配置策略如下：

<a id="tbl-credit-config"></a>
**表 10 Flow-Control Credit 配置策略**

| Credit 类型 | Synopsys 参数 | 配置策略 | 说明 |
| :--- | :--- | :--- | :--- |
| Posted Header/Data Credit | `RADM_PQ_HCRD_VC0` / `RADM_PQ_DCRD_VC0` | 使用 coreConsultant 自动计算值 | 用于接收 inbound MWr 等 Posted TLP；credit 与 posted receive queue buffer 绑定 |
| Non-Posted Header/Data Credit | `RADM_NPQ_HCRD_VC0` / `RADM_NPQ_DCRD_VC0` | 使用 coreConsultant 自动计算值，并满足表 11 的 NP header credit 校核目标 | 用于接收 inbound MRd/config/IO 等 NP TLP；NP data credit 主要覆盖带 payload 的 NP write，当前不是高吞吐主路径 |
| Completion Header/Data Credit | `RADM_CPLQ_HCRD_VC0` / `RADM_CPLQ_DCRD_VC0` | 使用 Completion Queue Management；不手工按表 11 直接填 credit | 用于接收本端 outbound MRd 返回的 Cpl/CplD；本项目 HDMA read 性能依赖 completion 接收能力 |
| Completion Queue Overflow Protection | `RADM_SEL_CPLQ_OVFLW_PRVNTN_VC` | `2` | RP/EP 采用该机制：Receiver advertises Infinite Header/Data Credits，并启用 Receive Completion Queue Management；当 completion storage 不足时 throttles outbound NP request |
| Completion Queue Management | `CX_CPLQ_MANAGEMENT_ENABLE` | `1` | 与 `RADM_SEL_CPLQ_OVFLW_PRVNTN_VC=2` 配套使用，用于管理本端 outbound NP request 对应的返回 completion storage |
| Flow-Control Scaling | `FC_SCALE_EN` 及 `RADM_*Q_*SCALE_VC0` | 使用 coreConsultant 自动计算值 | 当前 RTT=1000ns、MPS=256B 的校核目标未超过未缩放 credit 上限；如 coreConsultant 因 Gen5/link delay 自动启用 scaling，则采用其生成结果 |

表 11 给出按 Gen5、MPS=256B、RTT=1000ns 计算的 credit 校核目标。该表用于对 coreConsultant 生成配置进行项目吞吐目标覆盖检查，不作为直接手填 `RADM_*_HCRD/DCRD` 的配置值。

<a id="tbl-credit-estimation"></a>
**表 11 RTT=1000ns Credit 校核目标**

| Profile | Lane | 1000ns BDP | Posted Header Credit | Posted Data Credit | NP Header Credit | NP Data Credit | Completion Header Credit | Completion Data Credit | 配置结论 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| X4 profile | x4 | 15754B | 64 | 985 | 64 | CoreConsultant | CoreConsultant / CplQ Mgmt | 985 | VC0 credit 使用 coreConsultant 结果；HDMA read 配置 64 tags |
| X2 profile | x2 | 7877B | 32 | 493 | 32 | CoreConsultant | CoreConsultant / CplQ Mgmt | 493 | VC0 credit 使用 coreConsultant 结果；HDMA read 配置 32 tags |
| X1 profile | x1 | 3939B | 16 | 247 | 16 | CoreConsultant | CoreConsultant / CplQ Mgmt | 247 | VC0 credit 使用 coreConsultant 结果；HDMA read 配置 16 tags |

Credit 计算口径如下：

- Posted credit、Non-Posted credit 和 Completion credit 是三组独立的 PCIe flow-control credit pool；每组分别包含 Header credit 和 Data credit。
- HDMA write 发出 MWr TLP，消耗 Posted Header credit 和 Posted Data credit，不消耗 PCIe tag。
- HDMA read 发出 MRd TLP，MRd request 不带 data payload，消耗 NP Header credit，不消耗 NP Data credit；HDMA read 的数据通过远端返回 CplD，CplD 消耗 Completion Header credit 和 Completion Data credit。
- NP Data credit 主要用于带 payload 的 Non-Posted write，例如 I/O write、Configuration write 或 AtomicOp。本项目不使用 NP write 作为高吞吐主路径，因此 NP Data credit 使用 coreConsultant 自动计算值。
- Completion Header credit 受 CplD 拆包方式影响，拆包与 RCB、MRRS/MPS、远端 completer 行为和 Completion Queue Management 相关。本项目使用 `RADM_SEL_CPLQ_OVFLW_PRVNTN_VC=2` 和 `CX_CPLQ_MANAGEMENT_ENABLE=1`，Completion Header credit / buffer 由 coreConsultant 和 Completion Queue Management 处理。
- Posted Header credit 和 NP Header credit 的校核目标按 256B request/payload 粒度由 `ceil(BDP / 256B)` 计算，并按本项目 profile 配置取 64/32/16。
- Posted Data credit 和 Completion Data credit 的校核目标按 `ceil(BDP / 16B)` 计算。
- PCIe data credit 单位为 16B。
- Flit Mode 支持时，advertised header/data credit 必须满足 4-credit 对齐要求，由 coreConsultant 生成配置负责处理。

### 4.7 Synopsys Define / Parameter Configuration

<a id="tbl-snps-params"></a>
**表 12 Synopsys Define / Parameter 配置**

| 功能 | Synopsys Define / Parameter | X4 Profile | X2 Profile | X1 Profile | 说明 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 最大 lane 数 | `CX_NL` | 4 | 2 | 1 | Controller 最大 link width |
| Max Payload | `CX_MAX_MTU` | 256 | 256 | 256 | MPS=256B |
| HDMA enable | `CC_DMA_ENABLE` | 1 | 1 | 1 | 使能内部 DMA |
| Hyper DMA enable | `CX_DMA_PF_ENABLE` | 1 | 1 | 1 | 使用 HDMA |
| HDMA write channel | `CC_NUM_DMA_WR_CHAN` | 2 | 1 | 1 | H2D 方向 channel 数 |
| HDMA read channel | `CC_NUM_DMA_RD_CHAN` | 2 | 1 | 1 | D2H 方向 channel 数 |
| AXI Manager NP outstanding / ID | `CC_MAX_MSTR_TAGS_AXI` | 64 | 32 | 16 | AXI Manager 侧 NP outstanding / ID 资源规模 |
| AXI Subordinate NP tag | `CC_MAX_SLV_TAG` | 64 | 32 | 16 | AXI Subordinate outbound NP request outstanding 资源 |
| AXI Subordinate NP write set-aside | `CC_SLV_NUM_OUTSTND_CPU_WR_REQ` | 32 | 16 | 8 | AXI Subordinate NP write set-aside buffer 深度 |
| DMA read tag reserve | `CC_NUM_DMA_RD_TAG` | 64 | 32 | 16 | 从 tag pool 中保留给 HDMA MRd |
| Outbound NP tag pool | `CX_MAX_TAG` | 127 | 63 | 31 | 实际 tag 数为参数值 + 1 |
| Forwarded NP request LUT | `CX_REMOTE_MAX_TAG` | 127 | 63 | 31 | 与本 profile tag pool 对齐 |
| Completion queue overflow protection | `RADM_SEL_CPLQ_OVFLW_PRVNTN_VC` | 2 | 2 | 2 | Receiver advertises infinite Cpl credits，并启用 Completion Queue Management |
| Completion queue management | `CX_CPLQ_MANAGEMENT_ENABLE` | 1 | 1 | 1 | 管理 outbound NP request 对应返回 completion 的接收队列 |
| VC0 Posted RX credit | `RADM_PQ_HCRD_VC0` / `RADM_PQ_DCRD_VC0` | CoreConsultant | CoreConsultant | CoreConsultant | 使用 coreConsultant 自动计算值 |
| VC0 Non-Posted RX credit | `RADM_NPQ_HCRD_VC0` / `RADM_NPQ_DCRD_VC0` | CoreConsultant | CoreConsultant | CoreConsultant | 使用 coreConsultant 自动计算值，并满足表 11 校核目标 |
| VC0 Completion RX credit | `RADM_CPLQ_HCRD_VC0` / `RADM_CPLQ_DCRD_VC0` | CoreConsultant | CoreConsultant | CoreConsultant | 使用 Completion Queue Management，不按表 11 直接手填 |
| Flow-control scaling | `FC_SCALE_EN` / `RADM_*Q_*SCALE_VC0` | CoreConsultant | CoreConsultant | CoreConsultant | 使用 coreConsultant 自动计算值 |
| 10-bit tag | `CX_10BITS_TAG` | 1 | 1 | 1 | 启用 10-bit tag capability |
| 14-bit tag | `CX_14BITS_TAG` | 0 | 0 | 0 | 当前不使用 14-bit tag |
| SR-IOV enable | `CX_SRIOV_ENABLE` | 1 | 1 | 1 | 启用 SR-IOV |
| Internal VF 总数 | `CX_NVFUNC` | 8 | 4 | 2 | 由 `CX_MAX_VF_n` 求和 |
| PF0 VF 数量 | `CX_MAX_VF_0` | 8 | 4 | 2 | 每个 Controller 配置 1 个 PF |
| External VF | `CX_EXTENSIBLE_VFUNC` | 0 | 0 | 0 | 使用 internal VF |
| Dynamic VF allocation | `DYNAMIC_VF_ENABLE` | 0 | 0 | 0 | 使用 static VF allocation |
| VF stride always one | `CX_VF_STRIDE_ALWAYS_ONE` | 1 | 1 | 1 | VF Stride 固定为 1 |
| ARI enable | `CX_ARI_ENABLE` | 1 | 1 | 1 | SR-IOV 场景启用 ARI capability |
| MSI-X capability | `MSIX_CAP_ENABLE` | 1 | 1 | 1 | PF 支持 MSI-X |
| Integrated MSI-X | `MSIX_TABLE_EN` | 1 | 1 | 1 | 使用 Controller 内部 MSI-X table/PBA |
| PF MSI-X table size | `MSIX_TABLE_SIZE_0` | 15 | 7 | 3 | 字段值为 PF entry 数 - 1 |
| VF MSI-X capability | `VF_MSIX_CAP_ENABLE` | 1 | 1 | 1 | VF 支持 MSI-X |
| VF MSI-X table size | `[TBD]` | 3 | 1 | 1 | 字段值为每 VF entry 数 - 1；CoreConsultant 参数名为 `[TBD]` |
| iATU enable | `CX_INTERNAL_ATU_ENABLE` | 1 | 1 | 1 | 使用 internal iATU |
| outbound iATU region | `CX_ATU_NUM_OUTBOUND_REGIONS` | 8 | 4 | 2 | OB region 数 |
| inbound iATU region | `CX_ATU_NUM_INBOUND_REGIONS` | 8 | 4 | 2 | IB region 数 |
| FLR enable | `CX_FLR_ENABLE` | 1 | 1 | 1 | SR-IOV=1 时 FLR 支持由 SR-IOV 配置派生 |

---

## 5. Block Description

### 5.1 sys_pcie

`sys_pcie` 为 PCIe Controller 相关逻辑的上层集成 block，用于承载 PCIe Controller instance、Controller 侧配置/状态接口、AXI Manager/AXI Subordinate 接口、MSI/MSI-X/interrupt 相关接口、reset/clock 控制接口以及与 HSIO bus/PHY 侧的连接逻辑。

<a id="fig-sys-pcie"></a>**图 5 sys_pcie 结构图**

![图 5 sys_pcie 结构图](./images/sys_pcie_diagram.png)

`sys_pcie` 内部模块划分、寄存器窗口、中断汇聚、reset/clock 信号边界和与 `sys_hsio_pcie` / `sys_hsio_hybrid` 的接口关系为 `[TBD]`。

### 5.2 sys_hsio_pcie

`sys_hsio_pcie` 使用 `PHY_PCIE`，支持 C0 X4 独占模式或 C0 X2 + C1 X2 bifurcation 模式。

Lane 分配见 [表 2](#tbl-pcie-lane)。

### 5.3 sys_hsio_hybrid

`sys_hsio_hybrid` 使用 `PHY_HYBRID`，支持 PCIe X2 + 2 路 Ethernet 或 PCIe X1 + PCIe X1 + 2 路 Ethernet。

Lane 分配见 [表 3](#tbl-hybrid-lane)。

### 5.4 NOC / Bus

多个 PCIe Controller 和 10G Ethernet Controller 共享同一条 `hsio_bus` 上行通路。所有 Controller 的上行 DMA 通路接到 `hsio_bus` 同一个 slave-side 汇聚口，经过 `hsio_bus` 仲裁后，从同一个 master-side 出口进入 `hsio_sub_bus`。`hsio_sub_bus` 再根据地址将事务路由至 MNOC 或 SNOC。

PCIe Controller AXI Manager 接口规格为 `256bit @ 1GHz`，Ethernet manager 接口规格为 AXI 128-bit，`hsio_bus` 上行主干为 AXI 256-bit。由于多个 Controller 共享同一个上行汇聚口和同一个 master-side 出口，系统吞吐和延迟由 `hsio_bus` 仲裁、`hsio_sub_bus` 路由、MNOC/SNOC 目标带宽、NoC/DDR QoS 和背压共同决定。

NoC / Bus 需要覆盖以下并发场景：

- C0 X4 Gen5 独占 `PHY_PCIE` 时的 HDMA 单向/双向吞吐。
- 所有 active PCIe Controller 和 Ethernet Controller 同时工作时，`hsio_bus` 上行仲裁需保证各 Controller 获得符合 QoS 配置的服务，避免任一 Controller 长时间无法获得上行带宽。
- RC mode 下 config/PIO 访问通过 CPU/SNOC 下行访问 PCIe Controller AXI Subordinate，并进一步由 Controller 发起 PCIe config/MMIO TLP；该路径不经过 `hsio_bus` 上行 DMA 汇聚口。若同一 PCIe Controller 同时存在 HDMA 和 config/PIO 访问，需要在 Controller 内部仲裁、PCIe link 带宽、Non-Posted tag/credit 和 completion timeout 约束下保证 config/PIO 可完成。
- `hsio_sub_bus` 向 MNOC AXI 256-bit 与向 SNOC AXI 64-bit 同时访问时的仲裁和背压。

### 5.5 APR

APR 用于 PCIe 和 Ethernet partial reset 控制，使各个 Controller 可独立复位，并在异常场景下实现故障隔离。APR 支持单 Controller 复位、局部链路恢复和异常状态清除，避免影响其他正常工作的 PCIe/Ethernet link。

### 5.6 CKM

CKM 用于监测 PHY PLL 输出时钟稳定性，并向复位/初始化流程提供时钟稳定状态。监测点、稳定判定条件、异常上报和复位联动策略为 `[TBD]`。

### 5.7 AHB Backdoor

AHB backdoor 通路用于访问 PHY 内部 SRAM，在 PHY 启动之前 load firmware。该路径属于 PHY bring-up 和 firmware load 相关后门访问路径。

### 5.8 SYS-ETH

系统包含两个 10G Ethernet Controller，位于 `sys_hsio_hybrid`。

<a id="tbl-eth-config"></a>
**表 13 Ethernet Controller 配置**

| Controller | PHY/Lane | AXI 位宽 | AXI Outstanding | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| ETH0 | `PHY_HYBRID` Lane2 | AXI 128-bit | 10 | 10G/5G Ethernet |
| ETH1 | `PHY_HYBRID` Lane3 | AXI 128-bit | 10 | 10G/5G Ethernet |

Ethernet manager 是 Ethernet 的 DMA 口，通过异步边界接入 `hsio_bus`，并与 PCIe 共享 NoC/DDR 资源。Ethernet AXI outstanding 基于 AXI 域 BDP 预算，单 request 有效数据为 128B，RTT 为 1000ns。

---

## 6. Open Items

- [ ] PHY 初始化与配置流程。
- [ ] Bifurcation 模式编码、寄存器配置及非法组合处理。
- [ ] `sys_pcie_eth_wrap` harden partition 边界、跨 boundary 信号、物理约束和 floorplan 约束。
- [ ] `sys_pcie` 内部模块划分、寄存器窗口、中断汇聚、reset/clock 信号边界和 HSIO 接口关系。
- [ ] Common Clock / SRIS 模式、参考时钟分发、冷复位/热复位/partial reset 释放顺序。
- [ ] ADB master/slave、TBU 与 `hsio_sub_bus` 的连接关系。
- [ ] NoC/DDR QoS、仲裁策略、背压策略和 starvation 控制。
- [ ] IO 接口信号定义与中断映射。
- [ ] 低功耗策略，包括电源域、唤醒源、ASPM L0s/L1 进入/退出流程。
- [ ] 功能安全机制，包括 ECC、CRC、Data Protect、错误聚合和错误上报。
- [ ] Synopsys PCIe Gen5 Controller 的 HDMA descriptor、buffer depth、BAR、AER、ASPM、ECAM 等参数配置。
- [ ] VF BAR queue/doorbell/status window、HDMA interrupt 隔离和软件资源规划。

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
| 1.23 | 2026-07-22 | 修正 NoC / Bus 并发场景描述：区分 HDMA/Ethernet 上行汇聚路径与 RC config/PIO 下行及 PCIe outbound 路径；补充 PIO 定义 |
| 1.24 | 2026-07-22 | 将 NoC / Bus 公平性场景调整为所有 active PCIe Controller 和 Ethernet Controller 同时工作，覆盖共享 `hsio_bus` 汇聚口的整体仲裁行为 |
| 1.25 | 2026-07-22 | 删除 NoC / Bus 并发场景中与全体 active Controller 并发场景重叠的 C0 X2 + C1 X2 局部组合 |
| 1.26 | 2026-07-22 | 按 databook 更新 flow-control credit 配置策略：VC0 credit 使用 coreConsultant 自动计算，Completion Queue 使用 management/overflow protection，并保留 RTT=1000ns 校核目标 |
| 1.27 | 2026-07-22 | 按图片占位文件名更新 Clock/Reset 图引用；新增 harden 切分章节和 `module_division.png`；新增 `sys_pcie` block 描述和 `sys_pcie_diagram.png` 占位图 |
| 1.28 | 2026-07-22 | 将表 11 中 CplD / Posted Data Credit 表述改为 Payload Data Credit 等效校核目标，并补充该列的计算口径和适用路径 |
| 1.29 | 2026-07-22 | 重构表 11，分列列出 Posted/Non-Posted/Completion 的 Header/Data credit 校核目标，并明确 HDMA read data 与 HDMA write data 使用不同 credit pool |

---

**文档状态**：草稿，后续按 `[TBD]` 条目继续补充。
