# 乘法器设计文档

## 版本日志

| 版本 | 日期 | 作者 | 修改说明 |
|------|------|------|----------|
| 1.3  | 2026-03-11 | 文档助手 | 完善流水线设计描述，明确regslice使用full mode |
| 1.2  | 2026-03-11 | 文档助手 | 增补Wallace树压缩每bit运算细节 |
| 1.1  | 2026-03-11 | 文档助手 | 删除状态机描述部分 |
| 1.0  | 2026-03-10 | 文档助手 | 初始版本创建 |

## 术语介绍

- **AXI4-Stream**: AMBA AXI-Stream协议第四版，用于点对点数据传输
- **定点数**: 固定小数点数表示法，包含整数部分和小数部分
- **流水线级数**: 乘法运算划分为多个阶段，每阶段完成部分计算
- **TDATA**: AXI-Stream数据总线，承载有效数据载荷
- **TVALID/TREADY**: AXI-Stream握手信号，控制数据传输节奏
- **TLAST**: 数据包边界指示信号
- **TKEEP**: 字节有效指示信号
- **TSTRB**: 字节类型指示信号（数据字节/位置字节）

## 概述介绍

本设计实现一个支持AXI4-Stream协议的定点数乘法器，输入位宽64bit，支持流水线级数可配置。设计采用全流水线架构，能够连续接收输入数据并输出乘法结果，适用于高吞吐量数据处理场景。

设计核心特性：
- 兼容AXI4-Stream协议标准
- 支持有符号/无符号定点数乘法
- 输入数据格式：64bit (32bit整数部分 + 32bit小数部分)
- 输出数据格式：128bit (64bit整数部分 + 64bit小数部分)
- 可配置流水线级数：2-8级
- 全双工数据传输，支持背压控制

## 功能描述

### 支持特性

1. **协议兼容性**
   - 完全兼容AXI4-Stream协议规范
   - 支持标准握手信号（TVALID/TREADY）
   - 支持数据包边界指示（TLAST）
   - 支持字节有效指示（TKEEP）
   - 支持流标识符（TID/TDEST）

2. **数据格式支持**
   - 64bit输入数据宽度
   - 128bit输出数据宽度
   - 支持有符号定点数（2's补码表示）
   - 支持无符号定点数
   - 可配置小数点位宽（Q格式）

3. **流水线架构**
   - 可配置流水线级数：2, 4, 8级
   - 每级流水线寄存器可配置使能
   - 支持流水线暂停和刷新

4. **性能特性**
   - 最高工作频率：500MHz (典型工艺)
   - 吞吐量：1结果/周期（流水线满载时）
   - 延迟：可配置（2-8周期）

5. **可配置性**
   - 流水线级数通过parameter配置
   - 数据格式通过parameter选择
   - 握手模式可配置（阻塞/非阻塞）

### 功能介绍

#### 数据流处理流程

1. **输入接口阶段**
   - 接收AXI4-Stream输入数据
   - 解析64bit定点数（操作数A和操作数B）
   - 校验数据有效性（TKEEP/TSTRB）
   - 缓存输入数据至流水线第一级

2. **乘法运算阶段**
   - 定点数符号扩展处理
   - 部分积生成（基于Booth编码）
   - 部分积累加（Wallace树结构）
   - 最终结果累加和格式化

3. **流水线控制阶段**
   - 流水线寄存器使能控制
   - 数据冒险检测和处理
   - 流水线暂停和刷新机制

4. **输出接口阶段**
   - 128bit结果格式化
   - AXI4-Stream协议封装
   - 握手信号生成
   - 数据包边界标记

#### 协议交互

- **输入接口**: 支持标准AXI4-Stream握手，TVALID由上游驱动，TREADY由本模块根据内部状态驱动
- **输出接口**: 同样支持标准AXI4-Stream握手，TVALID由本模块根据计算结果驱动
- **背压传递**: 当输出接口无法接收数据时，背压会传递至输入接口
- **数据包处理**: TLAST信号标记数据包边界，保证数据包完整性

#### 错误处理

- **协议错误检测**: 检测非法的信号组合（如TVALID=1且TKEEP=0）
- **数据溢出检测**: 检测乘法结果溢出情况
- **配置错误检测**: 检测不合理的parameter配置

## 微架构描述

### 接口定义

#### 全局信号

| 信号名称 | 方向 | 位宽 | 描述 |
|----------|------|------|------|
| ACLK | 输入 | 1 | 全局时钟，所有信号在上升沿采样 |
| ARESETn | 输入 | 1 | 全局复位，低电平有效 |

#### AXI4-Stream输入接口

| 信号名称 | 方向 | 位宽 | 描述 |
|----------|------|------|------|
| s_axis_tvalid | 输入 | 1 | 输入数据有效，由上游驱动 |
| s_axis_tready | 输出 | 1 | 输入接口就绪，可接收数据 |
| s_axis_tdata | 输入 | 64 | 输入数据，包含两个32bit操作数 |
| s_axis_tstrb | 输入 | 8 | 字节类型指示，本设计固定为0xFF |
| s_axis_tkeep | 输入 | 8 | 字节有效指示，本设计固定为0xFF |
| s_axis_tlast | 输入 | 1 | 数据包边界指示 |
| s_axis_tid | 输入 | 8 | 流标识符 |
| s_axis_tdest | 输入 | 8 | 目标路由信息 |
| s_axis_tuser | 输入 | 16 | 用户自定义侧带信息 |

#### AXI4-Stream输出接口

| 信号名称 | 方向 | 位宽 | 描述 |
|----------|------|------|------|
| m_axis_tvalid | 输出 | 1 | 输出数据有效 |
| m_axis_tready | 输入 | 1 | 下游接收就绪 |
| m_axis_tdata | 输出 | 128 | 乘法结果数据 |
| m_axis_tstrb | 输出 | 16 | 字节类型指示，固定为0xFFFF |
| m_axis_tkeep | 输出 | 16 | 字节有效指示，固定为0xFFFF |
| m_axis_tlast | 输出 | 1 | 数据包边界指示，与输入同步 |
| m_axis_tid | 输出 | 8 | 流标识符，与输入一致 |
| m_axis_tdest | 输出 | 8 | 目标路由信息，与输入一致 |
| m_axis_tuser | 输出 | 16 | 用户自定义侧带信息，与输入一致 |

#### 配置接口（可选）

| 信号名称 | 方向 | 位宽 | 描述 |
|----------|------|------|------|
| cfg_pipe_stages | 输入 | 3 | 流水线级数配置(2,4,8) |
| cfg_signed_mode | 输入 | 1 | 有符号模式选择(0:无符号,1:有符号) |
| cfg_q_format | 输入 | 6 | Q格式配置，小数点位宽 |

### 顶层模块划分

```
Multiplier_top
├── AXI4S_Input_Interface
│   ├── 输入握手控制
│   ├── 数据缓存(FIFO)
│   └── 协议校验
├── FixedPoint_Multiplier_Core
│   ├── 操作数解析单元
│   ├── Booth编码器
│   ├── Partial_Product_Generator
│   ├── Wallace_Tree_Reducer
│   └── 最终累加器
├── Pipeline_Controller
│   ├── 流水线寄存器组
│   └── 冒险检测单元
├── AXI4S_Output_Interface
│   ├── 输出握手控制
│   ├── 结果格式化
│   └── 协议封装
└── Configuration_Register
    ├── 配置寄存器组
    └── 配置校验逻辑
```

#### 模块交互描述

1. **AXI4S_Input_Interface** 接收外部AXI4-Stream数据，进行协议检查后将有效数据传递给FixedPoint_Multiplier_Core
2. **FixedPoint_Multiplier_Core** 执行定点数乘法运算，生成中间结果
3. **Pipeline_Controller** 控制数据在流水线中的流动，处理冒险情况
4. **Pipeline_Registers** 根据配置的流水线级数插入寄存器，平衡时序
5. **AXI4S_Output_Interface** 将最终结果封装为AXI4-Stream格式输出


### 定点数乘法算法

#### 数据格式定义

- **输入操作数**: 64bit，格式为`{operand_b[31:0], operand_a[31:0]}`
- **定点数表示**: Qm.n格式，m为整数位，n为小数位
- **默认配置**: Q32.32格式（32位整数 + 32位小数）

#### Booth编码算法

采用基4 Booth编码，减少部分积数量：
- 每组3位进行编码：{b[i], b[i-1], b[i-2]}
- 生成部分积：0, ±A, ±2A
- 部分积数量从64个减少到33个

#### Wallace树压缩

Wallace树采用**3:2压缩器（全加器）**逐级减少部分积数量，最终通过**进位保留加法器**得到两个输出向量（和向量与进位向量），再经一次进位传递加法器产生最终结果。

**部分积对齐方式**
- 每个部分积宽度为128bit（64bit × 64bit乘法）
- 第i个部分积左移i×2位（基4 Booth编码，每次处理乘数2bit）
- 共33个部分积，各bit位按列对齐，形成33行×128列的位矩阵

**3:2压缩器（全加器）bit运算**
每个3:2压缩器输入三个相同权重的bit（a, b, c），输出和（sum）与进位（carry）：
```
sum  = a ^ b ^ c
carry = (a & b) | (b & c) | (c & a)
```
进位输出左移一位（即权重更高一位）送入下一级。

**压缩级详细bit运算过程**

| 级数 | 输入部分积数 | 输出部分积数 | 每列最大压缩器数 | 关键操作 |
|------|--------------|--------------|------------------|----------|
| 1    | 33           | 22           | 每列最多3个压缩器 | 对每列中≥3个的bit使用3:2压缩器，生成sum（同列）和carry（左移一列） |
| 2    | 22           | 15           | 每列最多2个压缩器 | 继续使用3:2压缩器，优先压缩列中bit数最多的列 |
| 3    | 15           | 10           | 每列最多2个压缩器 | 部分列已减少到≤2个bit，仅对≥3个bit的列进行压缩 |
| 4    | 10           | 7            | 每列最多1个压缩器 | 大部分列已为2个bit，少数列仍需压缩 |
| 5    | 7            | 5            | 每列最多1个压缩器 | 进一步压缩至5个部分积 |
| 6    | 5            | 3            | 每列最多1个压缩器 | 使用2:1压缩器（半加器）或3:2压缩器将5个减少到3个 |
| 7    | 3            | 2            | 无               | 使用进位保留加法器（3:2压缩器）将3个减少到2个（和向量S、进位向量C） |

**示例：第0列（最低有效位）bit运算**
假设第0列有3个bit：a₀, b₀, c₀（来自部分积0、1、2的第0位）
- 压缩器运算：
  ```
  sum₀  = a₀ ^ b₀ ^ c₀
  carry₁ = (a₀ & b₀) | (b₀ & c₀) | (c₀ & a₀)
  ```
- sum₀保留在第0列，carry₁送入第1列（权重更高一位）

**部分积压缩bit级流程图**
```
33个部分积 (128bit宽)
    ↓ 第1级压缩（每列独立）
22个中间向量
    ↓ 第2级压缩
15个中间向量  
    ↓ 第3级压缩
10个中间向量
    ↓ 第4级压缩
7个中间向量
    ↓ 第5级压缩
5个中间向量
    ↓ 第6级压缩
3个中间向量
    ↓ 第7级压缩（进位保留加法器）
和向量S[127:0], 进位向量C[127:0]
    ↓ 最终进位传递加法器
最终结果[127:0]
```

**进位保留加法器bit运算**
将最后3个向量A、B、D压缩为两个向量S和C：
- 对每一列i并行执行：
  ```
  sum_i = A[i] ^ B[i] ^ D[i]
  carry_i = (A[i] & B[i]) | (B[i] & D[i]) | (D[i] & A[i])
  ```
- S[i] = sum_i
- C[i+1] = carry_i（C[0] = 0）

**最终加法器bit运算**
将S和C相加得到最终128bit结果：
```
temp = S + (C << 1);
result = temp[127:0];
```
实际实现可使用超前进位加法器（CLA）或行波进位加法器（RCA），取决于时序要求。

**硬件资源消耗**
- 3:2压缩器总数：约 33×128/3 ≈ 1408 个全加器
- 布线复杂度：随部分积数量对数级增长
- 关键路径：最长进位链经过约 log₃(33) ≈ 3.5 级压缩器 + 最终加法器进位链

#### 结果处理

- 有符号数：进行符号扩展和补码处理
- 溢出处理：检测结果是否超出128bit表示范围
- 舍入处理：可配置的舍入模式（截断、四舍五入等）

### 流水线设计

本设计采用**全流水线架构**，通过插入**regslice**作为流水线寄存器，实现数据流的时序切割和握手解耦。流水线级数可配置为2、4、8级，每级流水线对应一个独立的regslice模块。

#### regslice模式选择

采用**full mode regslice**，同时寄存valid、data和ready信号，实现上下游完全解耦。full mode regslice内部包含一个深度为1的FIFO，支持背压传递和流水线暂停。

**full mode vs. forward/backward mode对比**

| 模式 | 寄存信号 | 内部结构 | 适用场景 | 本设计选择原因 |
|------|----------|----------|----------|----------------|
| **full mode** | valid + data + ready | 深度1 FIFO，完全握手解耦 | 高吞吐量、需要完全解耦的流水线 | 支持背压传递，允许上下游独立工作，最大吞吐量 |
| forward mode | valid + data | 寄存器链，ready直通 | 仅需时序切割，吞吐量要求不高 | 不适用，因为ready信号未寄存，背压响应延迟大 |
| backward mode | ready | 寄存器链，valid/data直通 | 仅需控制ready信号时序 | 不适用，因为valid/data未寄存，数据冒险处理复杂 |

#### 流水线级数配置

流水线寄存器插入位置如下：

| 流水线级数 | 插入位置 | 功能描述 |
|------------|----------|----------|
| **2级** | 1. 输入接口→乘法核心<br>2. 乘法核心→输出接口 | 两级简单流水线，时序切割最少 |
| **4级** | 1. 输入接口→Booth编码<br>2. Booth编码→Wallace树<br>3. Wallace树→最终累加<br>4. 最终累加→输出接口 | 均衡切割乘法运算关键路径 |
| **8级** | 在4级基础上，将Wallace树压缩的7级压缩过程每两级插入一级流水线 | 极致时序优化，适用于高频设计 |

#### regslice实现细节

**full mode regslice接口**
```systemverilog
module regslice_full #(
    parameter WIDTH = 128
) (
    input  wire             clk,
    input  wire             rst_n,
    // 上游接口
    input  wire             in_valid,
    output wire             in_ready,
    input  wire [WIDTH-1:0] in_data,
    // 下游接口
    output wire             out_valid,
    input  wire             out_ready,
    output wire [WIDTH-1:0] out_data
);
```

**内部FIFO控制逻辑**
- FIFO深度：1（单缓冲）
- 写使能：`fifo_push = in_valid && in_ready`
- 读使能：`fifo_pop = out_valid && out_ready`
- 满标志：`fifo_full = (state == FULL)`
- 空标志：`fifo_empty = (state == EMPTY)`

**状态机（简化）**
- **EMPTY**：FIFO空，`out_valid=0`，`in_ready=1`
- **FULL**：FIFO满，`out_valid=1`，`in_ready=0`
- 状态跳转：
  - EMPTY→FULL：`in_valid=1`且`in_ready=1`
  - FULL→EMPTY：`out_valid=1`且`out_ready=1`

#### 流水线控制策略

1. **数据冒险处理**
   - 由于full mode regslice的FIFO深度为1，最多只能缓冲一个数据
   - 当流水线下游阻塞时，背压通过`in_ready=0`传递至上游
   - 上游在`in_ready=0`时暂停数据发送，避免数据丢失

2. **流水线刷新**
   - 收到复位信号`ARESETn=0`时，所有regslice同步清空FIFO
   - `out_valid`立即置0，`in_ready`立即置1，恢复到EMPTY状态

3. **流水线旁路**
   - 当`ENABLE_PIPELINE_BYPASS=1`时，regslice可配置为直通模式
   - 直通模式下，`out_valid = in_valid`，`in_ready = out_ready`，`out_data = in_data`
   - 用于调试或低功耗场景，减少流水线延迟

#### 时序波形示例（full mode regslice）

参考`references/timing_waveform/full_mode_regslice.json`波形图：
- 上游发送3个flit（Flit0, Flit1, Flit2）
- regslice缓冲第一个flit，立即转发
- 当下游阻塞时（`out_ready=0`），regslice暂停转发，缓冲数据
- 下游恢复就绪后，继续转发缓冲数据

#### 性能影响分析

| 指标 | 无流水线 | 2级流水线 | 4级流水线 | 8级流水线 |
|------|----------|-----------|-----------|-----------|
| 最大频率 | 250MHz | 400MHz | 500MHz | 500MHz+ |
| 吞吐量 | 0.5结果/周期 | 1结果/周期 | 1结果/周期 | 1结果/周期 |
| 延迟 | 1周期 | 2周期 | 4周期 | 8周期 |
| 面积开销 | 0% | +5% | +10% | +20% |

**设计权衡**：
- **2级流水线**：面积开销最小，时序改善有限
- **4级流水线**：平衡时序和面积，推荐配置
- **8级流水线**：最大频率优化，面积开销最大，适用于超高频设计

#### 配置接口

通过`cfg_pipe_stages[2:0]`配置流水线级数：
- `3'b001`：2级流水线
- `3'b010`：4级流水线  
- `3'b100`：8级流水线
- 其他值：保留（默认4级）

### 可配置性设计

#### parameter定义

```systemverilog
// 流水线配置
parameter PIPELINE_STAGES = 4;    // 流水线级数：2,4,8
parameter ENABLE_PIPE_REG = 1;    // 流水线寄存器使能

// 数据格式配置
parameter SIGNED_MODE = 1;        // 1:有符号, 0:无符号
parameter Q_INT_BITS = 32;        // 整数位宽
parameter Q_FRAC_BITS = 32;       // 小数位宽

// 接口配置
parameter TDATA_WIDTH = 64;       // 输入数据位宽
parameter TDATA_OUT_WIDTH = 128;  // 输出数据位宽
parameter TID_WIDTH = 8;          // TID位宽
parameter TDEST_WIDTH = 8;        // TDEST位宽
parameter TUSER_WIDTH = 16;       // TUSER位宽

// 性能配置
parameter MAX_THROUGHPUT = 1;     // 最大吞吐量(结果/周期)
```

#### `define配置选项

```systemverilog
// 功能选择
`define ENABLE_BOOTH_ENCODING      // 使能Booth编码
`define ENABLE_WALLACE_TREE        // 使能Wallace树压缩
`define ENABLE_OVERFLOW_DETECTION  // 使能溢出检测
`define ENABLE_PIPELINE_BYPASS     // 使能流水线旁路

// 协议选项
`define AXI4S_FULL_PROTOCOL        // 完整AXI4-Stream协议
`define SUPPORT_TLAST_PROPAGATION  // 支持TLAST传播
`define SUPPORT_TKEEP_PROPAGATION  // 支持TKEEP传播
```

#### 配置寄存器映射

| 寄存器地址 | 名称 | 位宽 | 功能描述 |
|------------|------|------|----------|
| 0x00 | CTRL_REG | 32 | 控制寄存器 |
| 0x04 | CFG_PIPE_REG | 32 | 流水线配置 |
| 0x08 | CFG_DATA_REG | 32 | 数据格式配置 |
| 0x0C | STATUS_REG | 32 | 状态寄存器 |

### 时序设计

#### 关键时序路径

1. **输入接口到第一级流水线**: 包含握手逻辑和数据缓存
2. **Booth编码到部分积生成**: 组合逻辑路径较长
3. **Wallace树压缩**: 多级压缩器链
4. **最终累加到输出接口**: 包含结果格式化和协议封装

#### 时钟频率约束

- 目标频率: 500MHz (典型工艺下)
- 最坏情况路径: Wallace树压缩链
- 建立时间余量: 0.3ns
- 保持时间余量: 0.15ns

#### 波形图示例

以下为AXI4-Stream输入输出握手时序波形（使用wavedrom格式）：

```json
{
  "signal": [
    {"name": "ACLK", "wave": "P...."},
    {"name": "ARESETn", "wave": "01..."},
    {"name": "s_axis_tvalid", "wave": "0.1.0"},
    {"name": "s_axis_tready", "wave": "01.0."},
    {"name": "s_axis_tdata", "wave": "x.3.x", "data": ["operand_a,b"]},
    {"name": "s_axis_tlast", "wave": "0.1.0"},
    {},
    {"name": "m_axis_tvalid", "wave": "0..1."},
    {"name": "m_axis_tready", "wave": "01..0"},
    {"name": "m_axis_tdata", "wave": "x..4x", "data": ["result"]},
    {"name": "m_axis_tlast", "wave": "0..1."}
  ],
  "config": {"hscale": 2}
}
```

#### 流水线时序图

```json
{
  "signal": [
    {"name": "ACLK", "wave": "P......."},
    {"name": "Stage1", "wave": "0.1....0", "data": ["op_a,b"]},
    {"name": "Stage2", "wave": "0..1...0", "data": ["pp_gen"]},
    {"name": "Stage3", "wave": "0...1..0", "data": ["wallace"]},
    {"name": "Stage4", "wave": "0....1.0", "data": ["accum"]},
    {"name": "Output", "wave": "0.....10", "data": ["result"]}
  ],
  "config": {"hscale": 1.5}
}
```

### 面积估算

| 模块 | 逻辑门数 | 寄存器数 | 估算面积(um²) |
|------|----------|----------|---------------|
| 输入接口 | 1,200 | 256 | 12,000 |
| Booth编码器 | 3,500 | 128 | 18,000 |
| 部分积生成 | 8,000 | 512 | 40,000 |
| Wallace树 | 15,000 | 1,024 | 75,000 |
| 流水线控制 | 2,000 | 384 | 15,000 |
| 输出接口 | 1,500 | 320 | 12,000 |
| **总计** | **31,200** | **2,624** | **172,000** |

### 待办事项

1. [x] 完善Wallace树压缩器的具体实现细节
2. [ ] 添加更多舍入模式支持
3. [ ] 优化面积和功耗估算
4. [ ] 添加门级网表仿真验证
5. [x] 完善配置寄存器接口的详细描述
6. [ ] 添加低功耗设计（时钟门控）
7. [ ] 完善错误处理机制的详细设计
