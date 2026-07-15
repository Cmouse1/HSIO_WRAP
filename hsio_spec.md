# 技术规格书：HDD

**版本**：1.3
**状态**：草稿（初步版）

---

## 目录

- [Figure List](#figure-list)
- [Table List](#table-list)
- [1. Introduction](#1-introduction)
- [2. Feature List](#2-feature-list)
  - [2.1 PCIe 控制器特性](#21-pcie-控制器特性)
  - [2.2 PHY Bifurcation](#22-phy-bifurcation)
  - [2.3 SMMU](#23-smmu)
- [3. Functional Description](#3-functional-description)
  - [3.1 Architecture](#31-architecture)
    - [3.1.1 顶层结构](#311-顶层结构)
    - [3.1.2 组件互联](#312-组件互联)
    - [3.1.3 Bifurcation 架构](#313-bifurcation-架构)
    - [3.1.4 数据路径](#314-数据路径)
  - [3.2 Clock and Reset](#32-clock-and-reset)
  - [3.3 Design Description](#33-design-description)
  - [3.4 Matrix](#34-matrix)
    - [3.4.1 hsio\_bus](#341-hsio_bus)
    - [3.4.2 hsio\_sub\_bus](#342-hsio_sub_bus)
  - [3.5 Sub-IPs](#35-sub-ips)
    - [3.5.1 PCIe-Core](#351-pcie-core)
    - [3.5.2 32g-PHY](#352-32g-phy)
    - [3.5.3 APB-BUS](#353-apb-bus)
    - [3.5.4 APR](#354-apr)
    - [3.5.5 XAF](#355-xaf)
    - [3.5.6 ecc\_aggr](#356-ecc_aggr)
    - [3.5.7 CKM](#357-ckm)
    - [3.5.8 SYS-ETH](#358-sys-eth)
  - [3.6 IO Interfaces](#36-io-interfaces)
  - [3.7 Interrupt](#37-interrupt)
  - [3.8 Address Mapping](#38-address-mapping)
- [4. Block Description](#4-block-description)
  - [4.1 sys\_hsio\_pcie](#41-sys_hsio_pcie)
  - [4.2 sys\_hsio\_hybrid](#42-sys_hsio_hybrid)
  - [4.3 SMMU](#43-smmu)
  - [4.4 NOC 总线](#44-noc-总线)
  - [4.5 PCIe Controller](#45-pcie-controller)
    - [4.5.1 Architecture](#451-architecture)
    - [4.5.2 Initialization](#452-initialization)
    - [4.5.3 Functions](#453-functions)
- [5. 待补充信息清单](#5-待补充信息清单)
- [6. 变更记录](#6-变更记录)

## Figure List

- [图 1 顶层架构图](#fig-top-arch)

## Table List

- [表 1 sys_hsio_pcie 控制器 Lane 分配](#tbl-pcie-lane)
- [表 2 sys_hsio_hybrid 控制器 Lane 分配](#tbl-hybrid-lane)

---

## 1. Introduction

`sys_pcie_eth_pwr_wrap` 是一个多协议端口聚合子系统，包含：

- 4 个 PCIe Controller（1×X4 + 2×X2 + 1×X1）
- 2 个 10G Ethernet Controller
- 2 个 X4 32G-PHY（物理层接口）
- 1 个 SMMU
- NOC 总线

主要特性：通过 Bifurcation 技术实现多个控制器对单个 PHY 的 lane 复用。

---

## 2. Feature List

### 2.1 PCIe 控制器特性

**适用**：全部 4 个 PCIe Controller

- **速率**：支持 Gen1 / Gen2 / Gen3 / Gen4 / Gen5，向下兼容，自动协商
- **双模式**：支持 Dual Mode（EP & RC），启动时静态确定，不支持运行期间动态切换
- **SR-IOV**：需要启用，具体 PF/VF 数量待软件与产品需求确认
- **中断**：支持 MSI / MSI-X，支持 Legacy Interrupt
- **Resizable BAR**：可支持，默认不启用，除非 EP 大 BAR 或虚拟化场景需要
- **iATU**：支持内部地址转换单元（Internal Address Translation Unit）
- **FLR**：支持功能级复位
- **Partial Reset**：每个 PCIe Link 支持独立 partial reset
- **Hot Reset**：支持软件控制延迟恢复机制
- **Virtual Channel**：支持 1 个 VC
- **Max Payload Size**：支持 256B
- **DMA Engine**：使用 Controller 内部 DMA；通道数、outstanding 深度、buffer depth 和 descriptor 能力待 Synopsys databook 确认
- **功能安全**：支持，[TBD] 具体机制
- **AER**：支持高级错误上报
- **RAS DP**：支持 Data Protect 功能
- **ASPM**：仅支持 L0s 和 L1，不支持 L1.1 / L1.2 / L2
- **PCI-PM**：支持 PCI Power Management（D-state），链路可进入 L1
- **SRIS**：支持独立参考时钟
- **PTM**：可支持，默认不启用，除非系统需要精确时间同步
- **PIPE 接口**：支持 PIPE 4.4.1 规范
- **时钟结构**：PCLK 作为 PHY 输出时钟
- **热插拔**：不支持
- **ATS/PASID/PRS**：默认不启用，除非系统有 IOMMU/SVA/设备共享虚拟地址需求
- **AXI Manager 接口**：`256bit @ 1GHz`，作为 DMA 主通道
- **AXI Subordinate 接口**：`64bit @ 1GHz`，作为 CPU CSR/config/PIO 从通道，不承载高吞吐数据

### 2.2 PHY Bifurcation

- **PHY 数量**：2 个 X4 32G-PHY
- **单 Lane 速率**：32 Gbps
- **bifurcation 粒度**：支持 x4 / x2 / x1 lane 拆分配置
- **跨协议共享**：支持 PCIe 与 Ethernet 跨协议 PHY 共享
- **配置方式**：上电初始化时通过寄存器配置 bifurcation 模式
- **Static Bifurcation**：Lane 分配在初始化阶段静态确定，运行期间不可动态切换
- **PHY0 复用**：`sys_hsio_pcie` 的 X4 Controller 和 X2 Controller 通过静态 bifurcation mux 共享 PHY0；X4 使用 Lane0~Lane3，X4 Controller 在 X2 active mode 下使用 Lane0/Lane1，独立 X2 Controller 使用 Lane2/Lane3
- **PHY1 复用**：`sys_hsio_hybrid` 支持 PCIe X2 或 X1+X1 使用 Lane0/Lane1，同时两个 10G Ethernet 使用 Lane2/Lane3

### 2.3 SMMU

- **功能**：系统存储管理单元，为 PCIe Controller 提供 DMA 地址转换
- **转换粒度**：[TBD]
- **支持特性**：[TBD]

---

## 3. Functional Description

### 3.1 Architecture

`sys_pcie_eth_pwr_wrap` 子系统采用 **PHY-centric** 拓扑结构，以两个 X4 32G-PHY 为核心，通过 bifurcation 交换网络实现多个控制器对 PHY lane 的灵活分配。

#### 3.1.1 顶层结构

<a id="fig-top-arch"></a>**图 1 顶层架构图**

![图 1 顶层架构图](./images/top_architecture.png)

#### 3.1.2 组件互联

- **PCIe Controller**：4 个 PCIe Controller（`sys_hsio_pcie`：C0 max X4 + C1 max X2；`sys_hsio_hybrid`：C2 max X2 + C3 max X1）通过 bifurcation 连接到两个 PHY。控制器与 PHY 之间的 lane 映射在初始化阶段静态配置。
- **SMMU**：位于 PCIe Controller 与 MNOC 之间，为 PCIe DMA 事务提供统一的地址转换服务。
- **Ethernet Controller**：2 个 10G Ethernet Controller 固定连接到 `sys_hsio_hybrid` 的 Lane2/Lane3。
- **NOC 总线**：包含两条独立总线。`hsio_bus` 由 `snoc2hsio`（下行）和 `hsio2ring`（上行）两条隔离通路组成；`hsio_sub_bus` 将 `hsio2ring` 数据通路拆分为 `hsio2mnoc`（DDR 地址空间）和 `hsio2snoc`（非 DDR 地址空间）。

#### 3.1.3 Bifurcation 架构

**sys_hsio_pcie** 的 bifurcation 交换网络（纯 PCIe 域）：

- C0 PCIe X4 Controller 可使用 Lane0~Lane3。
- C0 PCIe X4 Controller 在 X2 active mode 下使用 Lane0/Lane1，并释放 Lane2/Lane3。
- C1 PCIe X2 Controller 使用 Lane2/Lane3。
- C0 与 C1 通过启动期静态 bifurcation mux 共享 PHY0；任意两个 active link 不得同时占用同一 Lane。具体 lane 分配见 [表 1](#tbl-pcie-lane)。

**sys_hsio_hybrid** 的 bifurcation 交换网络（PCIe + Ethernet 混合域）：

- Lane0/Lane1 可由 C2 PCIe X2 Controller 使用。
- Lane0/Lane1 也可拆分为 C2 PCIe X1 与 C3 PCIe X1。
- Lane2/Lane3 分别连接一个 10G Ethernet Controller。
- PCIe 与 Ethernet 共享 PHY1，PHY 内部使用不同 PLL。具体 lane 分配见 [表 2](#tbl-hybrid-lane)。

#### 3.1.4 数据路径

子系统包含以下主要事务路径：

- **系统下行访问**：SNOC 事务通过 `snoc2hsio` 进入 HSIO 子系统，并访问 PCIe、Ethernet 及公共控制寄存器。
- **PCIe DMA 上行访问**：PCIe Controller 发起的 DMA 事务经 SMMU 完成地址转换，再根据目标地址进入 `hsio2mnoc`（DDR）或 `hsio2snoc`（非 DDR）。
- **Ethernet 上行访问**：Ethernet Controller 发起的事务进入 HSIO 上行通路，并根据目标地址路由到 MNOC 或 SNOC。

各路径的数据位宽、时钟域跨越、QoS、Outstanding 数量及背压策略待补充。

### 3.2 Clock and Reset

时钟与复位架构待补充，至少需要明确：

- 各模块的时钟源、频率、门控方式与时钟域跨越。
- Common Clock / SRIS 模式及参考时钟分发关系。
- 冷复位、热复位、Partial Reset 的作用域、依赖关系与释放顺序。

### 3.3 Design Description

子系统按功能划分为 PHY 接入、协议控制、地址转换、片上互联和公共管理五部分：

- **PHY 接入**：两个 X4 32G-PHY 及 bifurcation mux。
- **协议控制**：4 个 PCIe Controller 和 2 个 10G Ethernet Controller。
- **地址转换**：SMMU，为 PCIe DMA 事务提供地址转换与隔离。
- **片上互联**：`hsio_bus` 与 `hsio_sub_bus`。
- **公共管理**：APB-BUS、APR、XAF、`ecc_aggr` 和 CKM。

### 3.4 Matrix

NOC 总线由两条独立的总线组成：

#### 3.4.1 hsio_bus

内部由两条完全隔离的通路构成：

- **snoc2hsio**：下行通路，负责系统 SNOC 到 HSIO 的数据传输
- **hsio2ring**：上行通路，负责 HSIO 到 ring_bus 的数据传输

#### 3.4.2 hsio_sub_bus

负责将 `hsio2ring` 的数据通路拆分为两部分：

- **hsio2mnoc**：地址范围为所有 DDR 空间
- **hsio2snoc**：地址范围为除去 DDR 地址空间外的其余所有地址空间

### 3.5 Sub-IPs

#### 3.5.1 PCIe-Core

PCIe-Core 采用 Synopsys PCIe Gen5 Controller，配置为 4 个独立 Controller instance：

| Controller | Max Lane | Active Mode | PHY/Lane | Role |
| :--- | :--- | :--- | :--- | :--- |
| C0 | X4 | X4 或 X2 | PHY0 Lane0~3 或 Lane0~1 | EP/RC boot-time dual mode |
| C1 | X2 | X2 | PHY0 Lane2~3 | EP/RC boot-time dual mode |
| C2 | X2 | X2 或 X1 | PHY1 Lane0~1 或 Lane0 | EP/RC boot-time dual mode |
| C3 | X1 | X1 | PHY1 Lane1 | EP/RC boot-time dual mode，是否保留完整能力可按产品需求收敛 |

所有 Controller 使用相同 AXI 接口规格：

- AXI Manager：`256bit @ 1GHz`，作为内部 DMA 主通道。
- AXI Subordinate：`64bit @ 1GHz`，作为 CPU CSR/config/PIO 从通道。

内部 DMA 能力、HDMA 类型、channel 数、descriptor 格式、outstanding 深度、tag 数、buffer depth 以及 EP/RC 模式下的功能差异待新版 Synopsys databook 确认。

#### 3.5.2 32g-PHY

[TBD]

#### 3.5.3 APB-BUS

[TBD]

#### 3.5.4 APR

[TBD]

#### 3.5.5 XAF

[TBD]

#### 3.5.6 ecc_aggr

[TBD]

#### 3.5.7 CKM

[TBD]

#### 3.5.8 SYS-ETH

[TBD]

### 3.6 IO Interfaces

[TBD]

### 3.7 Interrupt

[TBD]

### 3.8 Address Mapping

[地址映射表](./tables/address_mapping.xlsx)

---

## 4. Block Description

### 4.1 sys_hsio_pcie

**复用方式**：bifurcation（lane 拆分）

sys_hsio_pcie 的 lane 分配见下表。

<a id="tbl-pcie-lane"></a>
**表 1 sys_hsio_pcie 控制器 Lane 分配**

| 控制器 | 通道分配 | 说明 |
| :--- | :--- | :--- |
| C0 PCIe X4 Controller | Lane0~Lane3 或 Lane0~Lane1 | X4 active mode 使用 PHY0 全部 Lane；X2 active mode 使用 Lane0/Lane1，并释放 Lane2/Lane3 |
| C1 PCIe X2 Controller | Lane2~Lane3 | 使用 PHY0 Lane2/Lane3；可与 C0 的 X2 active mode 同时使用 |

### 4.2 sys_hsio_hybrid

**复用方式**：bifurcation + 跨协议共享（PCIe + Ethernet）

sys_hsio_hybrid 的 lane 分配见下表。

<a id="tbl-hybrid-lane"></a>
**表 2 sys_hsio_hybrid 控制器 Lane 分配**

| 控制器 | 通道分配 | 说明 |
| :--- | :--- | :--- |
| C2 PCIe X2 Controller | Lane0~Lane1 或 Lane0 | X2 active mode 使用 Lane0/Lane1；X1 active mode 使用 Lane0 |
| C3 PCIe X1 Controller | Lane1 | 在 X1+X1 PCIe 拆分模式下使用 |
| 10G ETH Controller #0 | Lane2 | XGMII-10G/5G |
| 10G ETH Controller #1 | Lane3 | XGMII-10G/5G |

### 4.3 SMMU

**功能**：系统存储管理单元，为 PCIe Controller 提供 DMA 地址转换

[TBD] 详细描述

### 4.4 NOC 总线

NOC 总线的通路划分与地址路由见 [§3.4 Matrix](#34-matrix)。

多个 PCIe Controller 和 10G Ethernet Controller 共享同一组 NoC/DDR 资源。PCIe Controller 侧 AXI Manager 接口规格为 `256bit @ 1GHz`，但系统是否能够达到 Gen5 满速取决于 NoC 总带宽、DDR 带宽、QoS、仲裁策略、DMA outstanding 深度和同时打流场景。

需要重点评估以下场景：

- C0 X4 Gen5 独占 PHY0 时的 DMA 单向/双向吞吐。
- C0 X2 + C1 X2 同时工作时的总带宽和公平性。
- C2/C3 PCIe 与两路 10G Ethernet 同时打流时的 QoS 和 starvation 风险。
- RC mode 下配置访问、PIO 访问与 DMA/ETH 大流量并发时的延迟。

### 4.5 PCIe Controller

#### 4.5.1 Architecture

PCIe Controller 采用 4 个完全独立的 instance。每个 Controller 具有独立 LTSSM、配置空间、复位、时钟、PERST# 处理、中断和 DMA 资源。

各 Controller 的 lane 连接关系如下：

- C0：max X4，可在 PHY0 Lane0~3 上作为 X4 link 工作；也可在 X2 active mode 下仅使用 Lane0/Lane1。
- C1：max X2，固定使用 PHY0 Lane2/Lane3。
- C2：max X2，可使用 PHY1 Lane0/Lane1；在 X1 active mode 下使用 Lane0。
- C3：max X1，使用 PHY1 Lane1。

当 C0 以 X2 active mode 工作时，PHY0 Lane2/Lane3 释放给 C1。C0 X4 active mode 与 C1 X2 active mode 互斥。

#### 4.5.2 Initialization

PCIe role、lane width、bifurcation mode 和 speed capability 均在启动阶段静态确定。系统不要求运行期间在 EP/RC 之间动态切换，也不要求运行期间动态改变 PHY lane 分配。

初始化阶段至少需要完成：

- 采样或配置 EP/RC role select。
- 配置 PHY0/PHY1 bifurcation mode。
- 等待各 Controller 独立 refclk/reset/PERST#/PLL lock 满足释放条件。
- 配置 Gen5/Gen4/Gen3/Gen2/Gen1 speed capability 和目标 link width。
- 配置 AXI、iATU/BAR、MSI/MSI-X、DMA 和错误上报相关寄存器。

#### 4.5.3 Functions

基础功能需求：

- 支持 Gen1/Gen2/Gen3/Gen4/Gen5，向下兼容并自动协商。
- 支持 Endpoint 和 Root Complex/Root Port boot-time dual mode。
- 支持 MSI/MSI-X，Legacy Interrupt 是否保留待软件需求确认。
- 支持 iATU、FLR、AER 和 completion timeout 相关能力。
- 使用 Controller 内部 DMA，经 AXI Manager 口访问共享 NoC。
- CPU 通过 AXI Subordinate 口访问 CSR/config/PIO。

默认不启用或待需求确认的功能包括 ARI、ATS/PASID/PRS、Resizable BAR、PTM、IDE、DPC/eDPC、NPEM、DPA、OBFF、Atomic Operation、TPH 和 TLP Prefix。SR-IOV 为明确需求，需要在 databook 到位后确认 PF/VF 数量、VF BAR、MSI-X 和软件资源规划。

---

## 5. 待补充信息清单

- [ ] PHY 初始化与配置流程
- [ ] Bifurcation 模式编码、寄存器配置及非法组合处理
- [ ] 各数据路径的位宽、时钟域、QoS、Outstanding 数量与背压策略
- [ ] SMMU 转换粒度、Stream ID 分配、旁路模式与异常处理
- [ ] IO 接口信号定义与中断映射
- [ ] 低功耗策略（电源域、唤醒源）
- [ ] 功能安全具体实现（ECC、CRC、lockstep）
- [ ] 性能指标（吞吐、延迟）
- [ ] Synopsys PCIe Gen5 Controller databook 到位后确认 HDMA、AXI、iATU、BAR、MSI/MSI-X、AER、ASPM、ECAM 等参数

---

## 6. 变更记录

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
| 1.3 | 2026-07-04 | 按当前项目需求更新：PHY0 支持 C0 X4 或 C0 X2+C1 X2，C1 使用 Lane2/Lane3；PHY1 支持 C2 X2 或 C2 X1+C3 X1，同时 Lane2/Lane3 连接 2 路 10G Ethernet；补充 boot-time dual mode、AXI M/S 和内部 DMA 需求 |

---

**文档状态**：初步版，待后续迭代补充。
