# PCIe Gen5 Controller Configuration Discussion

本文档记录当前项目对 Synopsys PCIe Gen5 Controller 的初步需求、已确认假设、配置倾向和后续 databook 到位后需要重点确认的项目。后续沟通可以直接在本文档中持续补充。

## 1. 项目背景

项目后续计划采用 Synopsys PCIe Gen5 Controller。license 和 databook 到位后，需要根据项目实际需求选择参考配置，并形成可用于内部评审、IP 配置和 AE 沟通的配置建议。

当前目标不是立即锁死所有参数，而是先明确系统拓扑、控制器实例、AXI/DMA 使用方式、bifurcation 能力和关键风险项。

## 2. 已确认的系统需求

### 2.1 PCIe Role

- PCIe Controller 需要支持 dual mode。
- dual mode 指 Endpoint 和 Root Complex/Root Port 均可支持。
- mode 在启动时确定。
- 不要求运行过程中 EP/RC 动态切换。

配置含义：

- 优先关注 boot-time role select。
- 需要确认 role select 信号、strap 或寄存器在 reset 阶段的稳定时序要求。
- 验证上需要覆盖 EP mode 和 RC mode，但不需要覆盖 link up 后动态切换 role。

### 2.2 Controller 和 PHY 拓扑

系统包含 2 个 x4 PHY，通过 bifurcation 供 PCIe 和 10G Ethernet 使用。

初步规划如下：

```text
PHY0: x4 PCIe PHY
  - C0: max x4 Controller
      - 可作为 x4 使用
      - 也可作为 x2 使用，并释放 lane2/3
  - C1: max x2 Controller
      - 使用 PHY0 lane2/3

PHY1: x4 shared PHY
  - lane0/1:
      - C2: max x2 Controller
      - 或拆分为 x1 + x1 PCIe 使用
  - lane2/3:
      - 两个 x1 10G Ethernet
```

最大同时形态理解为：

```text
PHY0 lane0/1: PCIe C0, x2 mode
PHY0 lane2/3: PCIe C1, x2 mode

PHY1 lane0: PCIe C2 or C2 x1
PHY1 lane1: PCIe C3 x1
PHY1 lane2: 10G Ethernet
PHY1 lane3: 10G Ethernet
```

### 2.3 PHY/Bifurcation 已确认事项

| 项目 | 当前结论 |
|---|---|
| PHY0 支持 `x4` 与 `x2+x2` 启动期 bifurcation | 已确认支持 |
| PHY1 支持 lane0/1 跑 `x2` 或 `x1+x1` PCIe，同时 lane2/3 跑两个 10G Ethernet | 已确认支持 |
| PCIe lane reversal / lane numbering / PIPE port mapping | 不受限，上一项目已使用过该 IP |
| 每个 bifurcated PCIe link 独立性 | 每个 Controller 完全独立 |
| refclk/reset/PERST/LTSSM | 每个 Controller 独立 |
| PCIe 和 10G Ethernet 共享 PHY | 支持，PHY 内部使用不同 PLL |
| Controller 小于最大 lane 数运行并释放未用 lane | 已确认支持 |

因此，当前最高优先级风险不再是 PHY bifurcation 是否可行，而是 Controller 配置、DMA/NoC 性能、EP/RC 软件资源和验证范围。

## 3. AXI 和 DMA 需求

### 3.1 AXI Interface

对 x4 / x2 / x1 Controller，AXI 配置保持一致：

| 接口 | 配置 | 用途 |
|---|---|---|
| AXI M | `256bit @ 1GHz` | DMA 主通道，高吞吐数据搬运 |
| AXI S | `64bit @ 1GHz` | CPU 从通道，用于 CSR/config/PIO |

当前判断：

- AXI M 理论带宽为 `32GB/s`，覆盖单个 Gen5 x4 单向吞吐需求基本足够。
- AXI S 理论带宽为 `8GB/s`，但只承载 CPU CSR/config/PIO，不承载高吞吐数据，因此配置合理。
- 多个 Controller 共享同一个 NoC，系统满速能力最终取决于 NoC/DDR/仲裁/QoS，而不只取决于 Controller AXI 位宽。

### 3.2 DMA

- 使用 Controller 内部 DMA。
- M 口是 DMA 主通道。
- 后续需要确认内部 DMA 是否在 EP 和 RC 两种模式下均可用。
- 如果所有 PCIe link 都有实际数据搬运需求，建议每个 Controller 保留独立 DMA 能力。

初步倾向：

| Controller | 建议 DMA 档位 | 说明 |
|---|---|---|
| C0 | High | 主性能 link，需支撑 x4 或高性能 x2 |
| C1 | Medium | x2 link，建议保留较完整 DMA 能力 |
| C2 | Medium | x2/x1 link，按实际吞吐需求配置 |
| C3 | Low/Medium | x1 link，可根据用途决定是否简化 |

注意：

- AXI 口位宽可以保持一致。
- DMA channel 数、outstanding 深度、buffer depth、descriptor 能力不一定需要完全一致。
- x1 Controller 如果按 x4 规格完整配置，可能带来不必要的面积和验证成本。

## 4. 旧版 Databook 截图信息

用户提供了上一版 PCI Express DM Controller Databook 截图，并说明新版 core 大概率与该版本相同。以下信息先作为参考假设，待新版 databook/license 到位后再逐项确认。

### 4.1 旧版 Databook 基本信息

- 截图来自 PCI Express DM Controller Databook 的 Product Overview / Features and Limitations。
- 截图页脚显示版本为 `6.21a`，日期为 `February 2025`。
- Gen5 and below 章节显示支持 PCI Express Base Specification Revision 6.2 的 non-optional features。
- Gen5 and below 支持 lane 配置包括 `x1 / x2 / x4 / x8 / x16`。

### 4.2 与本项目直接相关的能力点

| 能力项 | 从旧版截图得到的参考信息 | 对本项目的影响 |
|---|---|---|
| Lane width | Gen5 and below 支持 `x1 / x2 / x4 / x8 / x16` | 覆盖 C0 x4、C1/C2 x2、C3 x1 需求 |
| AXI datapath | Gen6 配置中出现 128/256/512/1024-bit internal datapath，并支持 AXI Manager/Subordinate 最高 1024-bit datapath | 当前 `256bit @ 1GHz` 属于保守配置，后续需确认 Gen5 配置页是否同样支持该组合 |
| AXI frequency | 截图提到 Single Port AXI 2GHz operation | 当前 `1GHz` 频率目标看起来有余量，但最终仍需以后端时序和交付配置为准 |
| AXI topology | 截图提到 Dual Port AXI Bridge，包含 2 Manager / 2 Subordinate | 本项目当前计划每 Controller 使用 1 个 M 口和 1 个 S 口；是否需要 dual port 暂无需求 |
| DMA | Gen6 截图中写到 Embedded DMA 仅支持 Hyper DMA (HDMA) | 若新版 Gen5/Gen6 core 也类似，需要重点确认内部 DMA 类型、是否为 HDMA、以及 EP/RC 两种模式下能力 |
| Tags | Gen6 配置中提到 14-bit Tags | 若 Gen5 配置也可选 extended tag/tag 扩展，需结合 DMA outstanding 和性能目标确认 |
| Flit/FEC | Gen6 支持 Flit Mode、FEC、64GT/s PAM4 等 | 本项目是 Gen5，通常不作为当前配置重点，但新版文档中若统一 core 需避免误开 Gen6-only 功能 |

### 4.3 旧版截图中出现的可选能力

截图中的 Gen5 and below optional features 包括以下类别，后续需要按项目实际需求决定是否打开：

| Feature | 初步建议 |
|---|---|
| SR-IOV / Extensible IOV | SR-IOV 为明确需求，需要开启；PF/VF 数量、VF BAR、MSI-X 和软件资源规划待确认 |
| ARI | 默认不开，除非需要大量 function 或配合 SR-IOV |
| ATS / PASID / PRS | 默认不开，除非系统有 IOMMU/SVA/设备共享虚拟地址需求 |
| Completion Timeout Ranges | 建议开启或保留标准能力，RC/EP 兼容性相关 |
| FLR | 建议开启，尤其 EP mode 对软件复位友好 |
| ID-Based Ordering / TPH / Atomic Operations / TLP Prefix | 默认按软件和互联需求决定，不建议盲目全开 |
| Dynamic Power Allocation / Emergency Power Reduction | 默认不开，除非平台电源管理明确需要 |
| L1 Substates / ASPM | ASPM 仅支持 L0s 和 L1；不支持 L1.1 / L1.2 / L2 |
| Extended Tag Support | 倾向开启，有利于 DMA outstanding 和高吞吐 |
| Resizable BAR / VF Resizable BAR | 默认不开；若 EP 暴露大 BAR 或虚拟化需求再考虑 |
| OBFF | 默认不开，除非平台软件和电源管理需要 |
| SRIS | 取决于 refclk 架构；如果系统存在独立 refclk/SSC 差异，需要重点确认 |
| Lightweight / Readiness Notifications | 默认不开，除非系统软件栈明确使用 |
| AER with Multiple Header Logging | 倾向开启 AER；Multiple Header Logging 视面积和调试需求决定 |
| VPD / Expansion ROM Validation | 默认不开，除非产品/固件生态需要 |
| PTM | 默认不开，除非系统需要精确时间同步 |
| ACS | RC mode 或多下游设备隔离时考虑开启；EP-only 场景通常不优先 |
| NPEM / DPC / eDPC | RC mode 下如果面向标准下游设备或高可靠场景可考虑；否则先作为可选项 |
| DMWr | 默认不开，除非软件和数据路径明确使用 |
| ECAM | RC mode 建议重点确认，可能影响软件枚举和 config space 访问方式 |
| IDE | 默认不开，除非产品有 PCIe 链路安全/加密需求 |

### 4.4 基于旧版 Databook 的初步判断

- 当前 x4/x2/x1 lane 需求与旧版 databook 展示的 lane 能力匹配。
- 当前 `256bit @ 1GHz` AXI M 口配置比较稳，不属于激进配置。
- 当前 `64bit @ 1GHz` AXI S 口作为 CPU CSR/config/PIO 通道合理。
- 高性能目标更依赖 DMA 类型、outstanding/tag、buffer depth、NoC/DDR 带宽和 QoS，而不是单纯 AXI 位宽。
- 需要特别确认新版 core 中 embedded DMA 是否就是 HDMA，以及 HDMA 对 EP/RC mode 的支持边界。

## 5. 初步 Controller 配置矩阵

| Controller | PHY/Lane | Max Lane | Active Mode | Role | AXI M | AXI S | DMA |
|---|---|---:|---|---|---|---|---|
| C0 | PHY0 lane0-3 or lane0-1 | x4 | x4 or x2 | boot-time EP/RC dual mode | 256b@1G | 64b@1G | High |
| C1 | PHY0 lane2-3 | x2 | x2 | boot-time EP/RC dual mode | 256b@1G | 64b@1G | Medium |
| C2 | PHY1 lane0-1 or lane0 | x2 | x2 or x1 | boot-time EP/RC dual mode | 256b@1G | 64b@1G | Medium |
| C3 | PHY1 lane1 | x1 | x1 | boot-time EP/RC dual mode, or按产品需求简化 | 256b@1G | 64b@1G | Low/Medium |

## 6. Gen5 性能目标建议

当前 Gen5 满速不是明确硬性指标，但希望尽量做到。

建议把目标拆成不同场景，而不是笼统要求所有组合都满速：

| 场景 | 建议目标 |
|---|---|
| 单个 C0 x4 独占 PHY0 | Gen5 x4 DMA 尽量接近满速，作为主性能目标 |
| C0 x2 + C1 x2 共用 PHY0 | 单 link 尽量接近 Gen5 x2；同时打流时评估 NoC/DDR 总带宽 |
| C2/C3 x1 PCIe | Gen5 x1 支持为主，吞吐目标按实际使用场景定义 |
| PCIe + 10G Ethernet 同时工作 | 重点验证 NoC 仲裁、QoS、无 starvation |
| RC mode | 重点关注枚举、配置访问、错误恢复和外设兼容性 |
| EP mode | 重点关注 DMA H2D/D2H 吞吐和中断路径 |

## 7. Databook 到位后需要确认的配置项

### 6.1 Dual Mode

- 是否支持 boot-time EP/RC role select。
- role select 是 strap、pin、寄存器还是 fuse 控制。
- role select 需要在哪个 reset 阶段稳定。
- EP mode 与 RC mode 下可用 feature 是否完全一致。
- dual mode 对面积、时序、寄存器空间和验证范围的影响。

### 6.2 Lane / Link / PIPE

- C0 是否可配置为 max x4，同时在 x2 active mode 下释放 lane2/3。
- C2 是否可配置为 max x2，同时支持 x1 active mode。
- 每个 Controller 的 PIPE interface width 和 lane enable 配置。
- link width negotiation、down-train 和 forced width 配置方式。
- Gen5/Gen4/Gen3/Gen2/Gen1 speed capability 配置方式。

### 6.3 DMA

- DMA 在 EP mode 和 RC mode 是否均可用。
- DMA channel 数可配置范围。
- scatter-gather / linked-list 支持情况。
- descriptor 格式和 alignment 要求。
- 64-bit address 支持。
- read/write 并发能力。
- outstanding request 数和 tag 数配置。
- AXI burst length、buffer depth、payload buffer 配置。
- DMA interrupt 支持 MSI/MSI-X/local interrupt 的方式。
- DMA 与 iATU/address translation 的关系。

### 6.4 AXI

- AXI M `256bit @ 1GHz` 是否为该 IP 推荐或支持配置。
- AXI S `64bit @ 1GHz` 是否满足 CSR/config/PIO。
- AXI outstanding ID、QoS、cache/prot/region 信号支持。
- AXI clock domain、CDC 和 reset 要求。
- 多 Controller 共享 NoC 时是否有推荐 QoS 设置。

### 6.5 Address Translation / BAR / Config Space

- EP mode BAR 数量、BAR size、64-bit BAR 支持。
- RC mode config access、ECAM 或 DBI 访问方式。
- iATU region 数量。
- inbound/outbound window 数量。
- 每个 Controller 的 DBI/CSR base、ECAM/config space base 规划。
- 32-bit/64-bit address window 支持。

### 6.6 Interrupt

- MSI/MSI-X 支持。
- INTx 是否需要。
- local interrupt 输出数量。
- DMA interrupt、AER/error interrupt、PME interrupt 路由。
- 多 Controller 中断聚合或独立接入 GIC/interrupt controller 的方式。

### 6.7 Error / Reliability / Debug

- AER 支持范围。
- ECRC、LCRC、replay buffer 相关配置。
- surprise down、link retrain、hot reset 支持。
- LTSSM 状态可观测性。
- link equalization/debug status。
- internal debug bus、trace、performance counter。

### 6.8 Power Management

- ASPM 仅支持 L0s 和 L1，不支持 L1.1 / L1.2 / L2；需确认 L0s/L1 的启用策略和退出时延。
- CLKREQ# 支持。
- PME 支持。
- D-state 支持。
- EP/RC mode 下低功耗行为差异。
- 与 PHY power state 的联动。

### 6.9 Reset / Clock / Strap

- PERST#、core reset、AXI reset、PHY reset 的时序要求。
- 每个 Controller 独立 reset 的集成方式。
- PHY shared PLL 与 PCIe/Ethernet 独立 PLL 的 reset/lock 时序。
- boot-time lane/mode/role/speed 配置采样时机。

## 8. 当前主要风险和关注点

| 风险/关注点 | 当前判断 |
|---|---|
| PHY bifurcation 可行性 | 已基本确认，风险较低 |
| Controller dual mode boot-time 配置 | 需要 databook 确认细节 |
| x4 Controller 在 x2 模式释放 lane | 已确认支持，但需在配置参数中落实 |
| 多 Controller 共享 NoC 后的满速能力 | 系统级风险，需要带宽预算和 QoS 规划 |
| DMA 参数是否足够支撑 Gen5 x4 | 需要 databook 确认 channel/outstanding/buffer/tag |
| C1/C2/C3 是否都需要完整 dual mode 和 DMA | 需要结合产品用例决定 |
| EP/RC 软件资源规划 | 需要提前规划 DBI/ECAM/BAR/iATU/interrupt |

## 9. 后续待补充信息

- 每个 Controller 的实际产品用途和优先级。
- C1/C2/C3 是否所有量产场景都需要 EP/RC dual mode。
- 每个 link 是否都需要 DMA，以及 DMA 吞吐目标。
- NoC 和 DDR 的总带宽预算。
- PCIe 与 10G Ethernet 同时打流时的 QoS 策略。
- 软件栈要求，例如 Linux RC driver、EP function、DMA driver、MSI/MSI-X 使用方式。
- Synopsys databook 中对应的具体配置参数名称和值域。

## 10. 当前推荐结论

当前建议按 4 个独立 PCIe Controller instance 规划：

- C0: max x4，支持 x4/x2，boot-time EP/RC dual mode，High DMA。
- C1: max x2，boot-time EP/RC dual mode，Medium DMA。
- C2: max x2，支持 x2/x1，boot-time EP/RC dual mode，Medium DMA。
- C3: max x1，是否完整 dual mode 和 DMA 档位可根据实际用途进一步收敛。

AXI 配置保持统一：

- AXI M: `256bit @ 1GHz`
- AXI S: `64bit @ 1GHz`

性能目标建议以 C0 x4 Gen5 DMA 为主目标，同时对多 Controller 并发场景做 NoC/DDR 带宽预算和 QoS 验证。
