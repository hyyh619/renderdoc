# RenderDoc DX11 (HLSL/DXBC) Shader 单步调试实现原理分析

> Session: `hlsl-interpreter` / step 1
> 日期: 2026-07-09
> 目标: 阅读源码，分析 RenderDoc 对 DX11 shader 指令的**单步跟踪、调试、变量查看**的具体实现原理，指明代码位置与主要函数。

---

## 目录

1. [一句话结论](#1-一句话结论)
2. [分析方法与执行过程（思考/执行）](#2-分析方法与执行过程思考执行)
3. [整体架构：CPU 解释器 + GPU 辅助](#3-整体架构cpu-解释器--gpu-辅助)
4. [核心数据模型](#4-核心数据模型)
5. [调试会话的建立（DebugVertex / DebugPixel / DebugThread）](#5-调试会话的建立)
6. [DXBC 字节码解释器核心](#6-dxbc-字节码解释器核心)
7. [GPU 辅助层：DebugAPIWrapper](#7-gpu-辅助层debugapiwrapper)
8. [单步状态生成与双向变化记录](#8-单步状态生成与双向变化记录)
9. [指令到源码/变量的映射](#9-指令到源码变量的映射)
10. [Replay Controller / Proxy 层](#10-replay-controller--proxy-层)
11. [Qt UI：单步驱动与变量查看](#11-qt-ui单步驱动与变量查看)
12. [端到端数据流总结](#12-端到端数据流总结)
13. [关键源码位置速查表](#13-关键源码位置速查表)

---

## 1. 一句话结论

RenderDoc 的 DX11 shader 调试**不是在 GPU 上单步执行**，而是：

- 在 **CPU 上用一个 DXBC 字节码解释器**（`DXBCDebug::InterpretDebugger` / `ThreadState`）逐条“解释执行”着色器的汇编指令；
- 对于 CPU 无法位精确（bit-exact）复现的操作（**纹理采样/gather、超越函数 rcp/rsq/exp/log/sincos、资源读取、资源元数据**），通过一个 **`DebugAPIWrapper` 回调接口**回到**真实 GPU/D3D11 设备**上跑一段极小的辅助 shader，把结果读回来；
- 每执行一条指令，解释器把**变量的前值/后值差量**记录成一个 `ShaderDebugState`；UI 侧把这些差量**正向/反向回放**，就实现了任意方向的单步与变量查看。

难点在于**输入的获取**：像素着色器的输入是光栅化插值出来的，无法直接读，RenderDoc 通过**重放该 draw、并临时替换成一个“抓取输入”的插桩像素着色器**，把目标像素的寄存器级输入写进 UAV 再读回。

---

## 2. 分析方法与执行过程（思考/执行）

**思考路径**：DX11 shader 调试涉及三层——(a) 面向 UI/Python 的 replay 控制器 API，(b) D3D11 驱动侧的调试入口与 GPU 辅助，(c) 与 API 无关的 DXBC 解释器内核。要讲清“单步/变量查看”，必须把这三层与其间的数据结构（`ShaderDebugTrace`/`ShaderDebugState`/`ShaderVariableChange`）串起来。

**执行步骤**：

1. 定位源码目录：
   - DX11 调试入口 → [renderdoc/driver/d3d11/d3d11_shaderdebug.cpp](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp)（2522 行）
   - DXBC 解释器 → [renderdoc/driver/shaders/dxbc/dxbc_debug.cpp](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp)（5741 行）+ [dxbc_debug.h](../renderdoc/driver/shaders/dxbc/dxbc_debug.h)
   - 公共数据模型 → [renderdoc/api/replay/shader_types.h](../renderdoc/api/replay/shader_types.h)
2. 直接精读解释器内核：`BeginDebug` / `ContinueDebug` / `StepNext` / `SetDst` / `GetSrc` / 控制流指令 / `CalcActiveMask` / `PrepareInitial` / `DDX`/`DDY`。
3. 精读数据结构 `ShaderDebugState`、`ShaderVariableChange`、`ShaderDebugTrace`、`SourceVariableMapping`。
4. 定位并阅读指令→源码行映射：`DXBCContainer::FillTraceLineInfo`（[dxbc_container.cpp:964](../renderdoc/driver/shaders/dxbc/dxbc_container.cpp#L964)）。
5. 并行派发两个探查子任务，分别深挖：
   - D3D11 调试入口与 `D3D11DebugAPIWrapper`（GPU 辅助采样/数学/资源读取）；
   - Replay Controller / Replay Proxy / Qt `ShaderViewer` 的单步驱动、正反向回放与变量面板。
6. 汇总为本报告。

---

## 3. 整体架构：CPU 解释器 + GPU 辅助

```
┌──────────────────────────────────────────────────────────────────────┐
│ Qt UI: ShaderViewer (qrenderdoc/Windows/ShaderViewer.cpp)              │
│  - 循环调用 ContinueDebug 累积 m_States                                 │
│  - 正向/反向应用 ShaderVariableChange 重建 m_Variables                  │
│  - 断点 / 当前指令高亮 / 变量&watch 面板                                 │
└───────────────▲───────────────────────────────────┬───────────────────┘
                │ rdcarray<ShaderDebugState>          │ Debug*/ContinueDebug
┌───────────────┴───────────────────────────────────▼───────────────────┐
│ IReplayController (api/replay/renderdoc_replay.h)                       │
│ ReplayController (replay/replay_controller.cpp)  —— 转发到驱动           │
│ [远程] ReplayProxy (core/replay_proxy.cpp) —— 序列化跨机                 │
└───────────────▲───────────────────────────────────┬───────────────────┘
                │                                     │
┌───────────────┴───────────────────────────────────▼───────────────────┐
│ D3D11Replay (driver/d3d11/d3d11_shaderdebug.cpp)                        │
│  - DebugVertex/DebugPixel/DebugThread：建立 ShaderDebugTrace + 输入      │
│  - D3D11DebugAPIWrapper：把 CPU 无法算的操作丢回真实 GPU                 │
└───────────────▲───────────────────────────────────┬───────────────────┘
                │ BeginDebug / ContinueDebug          │ FetchSRV/UAV, Sample, Math...
┌───────────────┴───────────────────────────────────▼───────────────────┐
│ DXBCDebug::InterpretDebugger / ThreadState (dxbc/dxbc_debug.cpp)        │
│  - 逐条解释 DXBC 指令，产出 ShaderDebugState 差量                        │
└────────────────────────────────────────────────────────────────────────┘
```

- **驱动无关内核**在 `DXBCDebug` 命名空间，D3D11 与 D3D12(SM5) 都复用它。
- 内核通过纯虚接口 `DXBCDebug::DebugAPIWrapper`（[dxbc_debug.h:145-176](../renderdoc/driver/shaders/dxbc/dxbc_debug.h#L145-L176)）与具体图形 API 解耦；D3D11 的实现是 `D3D11DebugAPIWrapper`。

---

## 4. 核心数据模型

文件：[renderdoc/api/replay/shader_types.h](../renderdoc/api/replay/shader_types.h)

| 结构 | 位置 | 作用 |
|------|------|------|
| `ShaderVariable` | [shader_types.h:330](../renderdoc/api/replay/shader_types.h#L330) | 一个变量的实际值（name/rows/columns/type + 联合体 value），可含 members（结构/数组）。是寄存器/输入/常量的统一载体。 |
| `ShaderVariableChange` | [shader_types.h:901-940](../renderdoc/api/replay/shader_types.h#L901-L940) | **一次变量变化的 `before` 与 `after`**。`before` 为空=该变量本步诞生；`after` 为空=该变量本步消亡。**双向可回放的关键。** |
| `ShaderDebugState` | [shader_types.h:946-1014](../renderdoc/api/replay/shader_types.h#L946-L1014) | **单步快照**：`nextInstruction`（下一条要执行的指令）、`stepIndex`（线性递增的程序计数）、`flags`（本步事件，如 NaN/Inf、采样）、`changes`（本步所有 `ShaderVariableChange`）、`callstack`。 |
| `ShaderDebugTrace` | [shader_types.h:1037-1132](../renderdoc/api/replay/shader_types.h#L1037-L1132) | **整个调试会话的不可变部分**：`inputs`、`constantBlocks`、`readOnlyResources`、`readWriteResources`、`samplers`、`sourceVars`、`instInfo`，以及**不透明句柄 `ShaderDebugger *debugger`**。 |
| `ShaderDebugger` | [shader_types.h:1019-1029](../renderdoc/api/replay/shader_types.h#L1019-L1029) | 只含虚析构的**不透明基类**；实际是 `InterpretDebugger`。Python/UI 只把它当句柄传回 `ContinueDebug`。 |
| `SourceVariableMapping` / `DebugVariableReference` | [shader_types.h:670](../renderdoc/api/replay/shader_types.h#L670) / [605](../renderdoc/api/replay/shader_types.h#L605) | 把 **HLSL 源码级变量**映射到一个或多个**调试变量**（寄存器/输入/常量的某几个分量）。变量面板据此把底层寄存器还原成源码变量。 |
| `InstructionSourceInfo` | [shader_types.h:860](../renderdoc/api/replay/shader_types.h#L860) | 每条指令的 `lineInfo`（源码行/列 + 反汇编行）与该指令处的局部 `sourceVars`。 |

**设计要点**：`ShaderDebugState` 只存**差量**而非整份寄存器快照，UI 侧维护一份运行中的变量集合并原地增量更新，因此内存与序列化开销都很小，且天然支持反向单步。

---

## 5. 调试会话的建立

文件：[renderdoc/driver/d3d11/d3d11_shaderdebug.cpp](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp)

三个入口都产出一个 `ShaderDebugTrace`，其中 `debugger` 指向一个新建的 `DXBCDebug::InterpretDebugger`，并填好 `inputs` / `constantBlocks`。

### 5.1 `D3D11Replay::DebugVertex`（[:1520-1883](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp#L1520)）—— 最简单

顶点输入来自顶点缓冲，可直接读，**无需重放**：
- 取输入布局 `GetLayoutDesc`（:1551），按目标 `vertid`/`instid` 与 per-instance step rate 计算偏移；
- 用 `GetDebugManager()->GetBufferData(...)` 直接读出该顶点/该实例的原始字节（:1584-1609）；
- `BeginDebug(dxbc, refl, 0)`（:1611），`AddCBuffersToGlobalState` 载入常量缓冲；
- 按 `ResourceFormat` 把原始字节解码进各输入寄存器 `ShaderVariable`（含 R10G10B10A2/R11G11B10/R5G6B5 等打包格式，:1699-1842），`SV_VertexID`/`SV_InstanceID` 直接填。

### 5.2 `D3D11Replay::DebugPixel`（[:1885-2376](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp#L1885)）—— 最难，核心技巧

像素着色器输入是光栅化器插值产物，无法直接读。做法是**重放该 draw + 插桩像素着色器**：

1. **快照管线**：`D3D11RenderStateTracker tracker(...)`（:1894），取 PS 及上游产几何的阶段（GS→DS→VS，:1904-1950）以获得喂给光栅器的 signature。
2. **配置 InputFetcher**：`DXDebug::InputFetcherConfig cfg` 设置目标像素 `cfg.x/cfg.y`、wave size = 4（一个像素 quad），保留一个 RTV/DSV 以维持正确的 MSAA 级别，UAV 槽取在 RT 之后；`CreateInputFetcher(...)`（:2000）**生成一段插桩 PS 的 HLSL**。
3. **编译插桩 PS**（:2002）：`MakePShader(fetcher.hlsl, "ExtractInputs", "ps_5_0")`。它做与真实 PS 相同的插值，但把目标像素每个 fragment 的**寄存器级输入**（插值值、导数、coverage、primitiveID、sample、frontface、helper 标志）写进结构化 UAV。
4. **建捕获缓冲**：`initialBuf`（命中记录，元素 0 是 `numHits` 计数）、可选 `evalBuf`（`EvaluateAttributeAtSample` 的按 sample 求值结果）、以及对应的 STAGING 回读缓冲与 UAV，全部清零（:2005-2110）。
5. **重放该 draw**：绑定插桩 PS + UAV + 保留的 RTV/DSV，`OMSetRenderTargetsAndUnorderedAccessViews(...)`，然后
   ```cpp
   m_pDevice->ReplayLog(0, eventId, eReplay_OnlyDraw);   // :2125 只重发该 draw
   ```
   于是相同几何被光栅化，目标像素真实输入落入 UAV，`CopyResource` 到 staging（:2112-2130）。
6. **读回命中并挑选“获胜” fragment**（:2132-2269）：一个像素可能被多个 fragment 覆盖，按应用的深度比较函数（或指定 `primitive`/`sample`）选出真正写入的那个 fragment。
7. **建立解释器**（:2282-2284）：
   ```cpp
   DXBCDebug::InterpretDebugger *interpreter = new DXBCDebug::InterpretDebugger;
   ShaderDebugTrace *ret = interpreter->BeginDebug(dxbc, refl, hit->quadLaneIndex);
   ```
   `quadLaneIndex` 告诉解释器 4 个 quad lane 里哪个是被调试像素。
8. **填 quad 4 lane 的输入**（:2292-2368）：把每 lane 的插值原始字（`memcpy`）填入输入寄存器；非活跃 lane 先拷活跃 lane 输入以保证导数（DDX/DDY）可用；按 sample 求值的输入存入 `global.sampleEvalCache`。

### 5.3 `D3D11Replay::DebugThread`（[:2378-2495](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp#L2378)）—— compute

无插值输入，只有 dispatch 的 ID：`BeginDebug(dxbc, refl, activeIndex)`，把 `SV_GroupID`/`SV_GroupThreadID`/`SV_DispatchThreadID`/`SV_GroupIndex` 作为“伪输入”直接算好填入（:2426-2492）。（`DebugMeshThread` 在 D3D11 不支持，返回空 trace。）

### 5.4 公共辅助 `AddCBuffersToGlobalState`（[:1497-1518](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp#L1497)）

对最多 14 个常量缓冲槽用 `GetBufferData` 读回字节，`AddCBufferToGlobalState` 填入 `global.constantBlocks`。三个入口都调用。

---

## 6. DXBC 字节码解释器核心

文件：[renderdoc/driver/shaders/dxbc/dxbc_debug.cpp](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp) + [dxbc_debug.h](../renderdoc/driver/shaders/dxbc/dxbc_debug.h)

### 6.1 三个核心类（[dxbc_debug.h](../renderdoc/driver/shaders/dxbc/dxbc_debug.h)）

- **`GlobalState`**（[:61-135](../renderdoc/driver/shaders/dxbc/dxbc_debug.h#L61)）：跨 lane 共享的全局态——SRV/UAV 数据（`std::map<BindingSlot, ...>`）、groupshared 内存、`constantBlocks`、`sampleEvalCache`。资源数据**按需**由 `DebugAPIWrapper::FetchSRV/FetchUAV` 填充。
- **`ThreadState`**（[:178-249](../renderdoc/driver/shaders/dxbc/dxbc_debug.h#L178)）：**一条 lane（一个 invocation）的执行态**——`nextInstruction`、`inputs`、`variables`（寄存器堆：temp/output/indexable temp 等）、`semantics`（线程/像素语义）。关键方法：`PrepareInitial`、`StepNext`、`SetDst`、`GetSrc`、`DDX`/`DDY`、`AssignValue`、`MarkResourceAccess`。
- **`InterpretDebugger`**（[:251-275](../renderdoc/driver/shaders/dxbc/dxbc_debug.h#L251)）：`ShaderDebugger` 的具体实现。持有 `GlobalState global`、`workgroup`（`rdcarray<ThreadState>`）、`activeLaneIndex`、`steps`。方法：`BeginDebug`、`ContinueDebug`、`CalcActiveMask`。

### 6.2 `InterpretDebugger::BeginDebug`（[:5065-...](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp#L5065)）

- 设置 `ret->debugger = this`，读取 `DCL_THREAD_GROUP` 得到 numthreads；
- **决定 workgroup 大小**（:5091）：**像素=4（quad），顶点=1，compute=1 或整组**（当使用 workgroup scope 且开启 `D3D_Hack_EnableGroups()`）；
- 为每条 lane 建 `ThreadState`，compute 下算出各 lane 的 `ThreadID`；
- compute 时 `global.PopulateGroupshared(...)`；
- 遍历输入 signature，为活跃 lane 建立输入 `ShaderVariable` 并生成 `SourceVariableMapping`（:5119-5192）。

### 6.3 单条指令执行：`ThreadState::StepNext`（[:2025-...](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp#L2025)）

这是解释器的心脏。流程（:2028-2052）：
1. 取当前指令 `op = program->GetInstruction(nextInstruction)`；
2. `apiWrapper->SetCurrentInstruction(nextInstruction)`；`nextInstruction++`（**默认顺序前进**，控制流指令再改写它）；
3. 若需要，取调用栈 `debug->GetCallstack(...)`；
4. **预取所有源操作数**：`for i in 1..operands: srcOpers.push_back(GetSrc(op.operands[i], op))`；
5. 一个**超大 `switch(op.operation)`** 分派到每个 opcode：
   - **算术**（`ADD/DIV/UDIV/...`）：如 `SetDst(state, op.operands[0], op, add(srcOpers[0], srcOpers[1], optype))`（:2060）；
   - **位运算**（`BFREV/COUNTBITS/FIRSTBIT_*`）；
   - **采样/资源**（`SAMPLE*/GATHER*/LD*/RESINFO/...`）→ 调 `apiWrapper->CalculateSampleGather/GetResourceInfo/...`；
   - **超越函数**（`RCP/RSQ/EXP/LOG/SINCOS`）→ 调 `apiWrapper->CalculateMathIntrinsic`；
   - **导数**（`DERIV_RTX/Y`）→ `DDX`/`DDY` 用 quad 邻居差分；
   - **控制流**（见 6.6）。

### 6.4 写目标：`ThreadState::SetDst`（[:1155-1294](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp#L1155)）—— **变量变化记录的关键**

- 解析目标操作数索引（含相对寻址），定位寄存器 `v`（temp/output/indexable temp/depth/coverage 等；写 input/cbuffer 报错，NULL 静默跳过）；
- 应用 `saturate`；
- **记录差量**：
  ```cpp
  ShaderVariableChange change = {*changeVar};      // :1258 先存 before
  ... AssignValue(*v, comp, right, ..., flushDenorm) ...  // 按写掩码逐分量赋值
  if (state) {
      state->flags |= flags;
      change.after = *changeVar;                    // :1291 再存 after
      state->changes.push_back(change);             // :1292 推入本步 changes
  }
  ```
- `AssignValue`（[:1129-1153](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp#L1129)）在赋值时检测非有限值置 `GeneratedNanOrInf` 标志，并对 denorm 做 flush。

> 正是 `SetDst` 在每次写寄存器时同时保存 `before`/`after`，才使 UI 能双向单步。

### 6.5 读源：`ThreadState::GetSrc`（[:1441-...](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp#L1441)）

从寄存器堆按操作数类型取值：`TYPE_TEMP`/`INDEXABLE_TEMP`/`OUTPUT`（:1469）、`TYPE_INPUT`（:1504，读 `inputs[]`）、资源/采样器/UAV/NULL（:1521，仅带回 binding 标识）、`IMMEDIATE32/64`（:1537，立即数）等，随后（函数后段）应用 swizzle 与 abs/neg 修饰、以及 float 输入的 denorm flush。

### 6.6 控制流：结构化跳转（[:4380-4637](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp#L4380)）

DXBC 是**结构化控制流**（无任意 goto），解释器靠**扫描配对标记**改写 `nextInstruction`：
- **`IF`**（:4537）：条件为假时向后扫描到配对的 `ELSE`/`ENDIF`（用 `depth` 计数嵌套）跳过 if 块；`ELSE`（:4581）跳到配对 `ENDIF`；
- **`SWITCH`**（:4380）：读 switch 值，扫描各 `CASE`/`DEFAULT` 找匹配位置跳转；
- **`LOOP`/`ENDLOOP`/`CONTINUE`/`CONTINUEC`**（:4457-4499）：向回扫描配对 `LOOP` 实现回跳；
- **`BREAK`/`BREAKC`**（:4501）：向前扫描到配对 `ENDLOOP`/`ENDSWITCH` 跳出；
- **`RET`/`RETC`**（:4625）与 **`DISCARD`**（:4610）：置 `done = true` 结束该 lane。

### 6.7 lane 活跃掩码与收敛：`InterpretDebugger::CalcActiveMask`（[:5452-...](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp#L5452)）

- 全部 lane 初始为活跃；只有 pixel/compute 需要发散/收敛管理；
- 若所有 lane `nextInstruction` 一致（lockstep）直接返回；
- **compute**：若有 lane 处于 `SYNC`（barrier），落后的 lane 先跑、到达 `SYNC` 的 lane 暂停，避免死锁（:5476-5509）；
- **pixel**：发散时不必 lockstep（此时用导数本就非法），收敛回同一指令时重新对齐，保证 DDX/DDY 有效（:5511+）。

### 6.8 导数：`DDX`/`DDY`（[:1385-1439](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp#L1385)）

用 quad 内邻居 lane 的 `GetSrc` 做差分：coarse 用左上像素邻居，fine 用直接左/上邻居。这解释了为何**像素调试必须以 4 lane 的 quad 为单位**。

---

## 7. GPU 辅助层：DebugAPIWrapper

抽象接口：[dxbc_debug.h:145-176](../renderdoc/driver/shaders/dxbc/dxbc_debug.h#L145)；D3D11 实现：`D3D11DebugAPIWrapper`（[d3d11_shaderdebug.cpp:46-92](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp#L46)）。

每次 `ContinueDebug` 在栈上构造一个 `D3D11DebugAPIWrapper`（持有 device、dxbc、`GlobalState&`、eventId）。当 CPU 解释器遇到无法忠实模拟的操作时回调它，用**真实 GPU**服务。其析构（:100-112）会在必要时把日志重放回 draw 之后以恢复状态一致性。

### 7.1 `FetchSRV`（[:120-192](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp#L120)）
按被调试阶段选出绑定的 SRV，填格式信息（`FillViewFmt` 或从反射查 typeless/structured 格式），对 buffer 类资源用 `GetDebugManager()->GetBufferData(...)` 把内容读入 `GlobalState::SRVData.data`。（纹理像素不在此批量拷贝，纹理读走 `CalculateSampleGather`。）

### 7.2 `FetchUAV`（[:194-421](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp#L194)）
UAV 可能被正在调试的 draw 写过，先**重放到 draw 之前**去脏（仅一次）：
```cpp
m_pDevice->ReplayLog(0, m_EventID, eReplay_WithoutDraw);   // :201
```
再取隐藏 append/consume 计数（`GetStructCount`）；buffer UAV 用 `GetBufferData` 读；**纹理 UAV** 则建 STAGING 副本、`CopyResource` 后 `Map` + `memcpy`（保留 row/depth pitch）。

### 7.3 `CalculateSampleGather`（[:1079-1419](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp#L1079)）—— GPU 上做纹理采样
`sample/gather/ld/lod` 依赖硬件过滤、寻址、格式转换，CPU 无法位精确复现。做法是**在 GPU 上跑一个 1×1 的全屏三角形 draw**：
- 取预建的 `ShaderDebugging` 辅助资源（`SampleVS`、按 texel offset 变体的 `SamplePS`、`ParamBuf`、`OutBuf`/`OutStageBuf`/`OutUAV`、`DummyRTV`，声明于 [d3d11_replay.h:109-131](../renderdoc/driver/d3d11/d3d11_replay.h#L109)）；
- 清洗 NaN/Inf，填 `DebugSampleOperation`（UV、导数、纹理维度、返回类型、gather 通道、sample index、LOD/compare、operation）到 `ParamBuf`；
- 绑定该 opcode 实际使用的 SRV + 采样器（`SAMPLE_B` 会克隆采样器改 `MipLODBias`；比较型放槽 1）；
- 设 1×1 视口，绑 `DummyRTV` + `OutUAV`，`context->Draw(3, 0)`（:1376），`CopyResource` 到 staging，`Map` 读回，按返回类型（float/uint/int 三份）与 `swizzle` 写入 `output`。

### 7.4 `CalculateMathIntrinsic`（[:1421-1495](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp#L1421)）—— GPU 上做超越函数
`rcp/rsq/exp/log/sincos` 为与硬件位精确，改用**compute dispatch**：填 `DebugMathOperation{mathOp, mathInVal}` → `CSSetShader(debugData.MathCS)` → `Dispatch(1,1,1)`（:1472）→ 读回两个 float4（`sincos` 需两输出）。

### 7.5 资源元数据（纯描述符读取，无 GPU dispatch）
- `GetSampleInfo`（:423）服务 `sampleinfo`（返回 `SampleDesc.Count`）；
- `GetBufferInfo`（:569）服务 `bufinfo`（返回 `NumElements`）；
- `GetResourceInfo`（:696）服务 `resinfo`（宽/高/深/数组层/mip 数，按 `mipLevel` 位移）。

---

## 8. 单步状态生成与双向变化记录

### 8.1 `ContinueDebug` 主循环（[dxbc_debug.cpp:5586-5658](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp#L5586)）

```cpp
rdcarray<ShaderDebugState> InterpretDebugger::ContinueDebug(DebugAPIWrapper *apiWrapper) {
    ThreadState &active = activeLane();
    if (active.Finished()) return {};                 // 空数组 = 调试结束

    if (steps == 0) {                                 // 首次：产出初始状态
        ShaderDebugState initial;
        active.PrepareInitial(initial);               // 把所有初始变量作为 before=空 的 change
        ret.push_back(std::move(initial)); steps++;
    }

    // 像素着色器保存旧 workgroup，保证跨 quad 的 DDX/DDY 结果一致
    for (int stepEnd = steps + 100; steps < stepEnd;) {   // 每次最多推进 ~100 步
        if (active.Finished()) break;
        CalcActiveMask(activeMask);                   // 计算哪些 lane 活跃
        for (int i = 0; i < workgroup.count(); i++) {
            if (!activeMask[i]) continue;
            if (i == activeLaneIndex) {
                ShaderDebugState state;
                workgroup[i].StepNext(&state, apiWrapper, oldworkgroup);  // 记录状态
                state.stepIndex = steps;
                state.nextInstruction = workgroup[i].nextInstruction;
                ret.push_back(std::move(state)); steps++;
            } else {
                workgroup[i].StepNext(NULL, apiWrapper, oldworkgroup);    // 只推进不记录
            }
        }
    }
    return ret;
}
```

要点：
- **只为 `activeLaneIndex` 记录 `ShaderDebugState`**；其余 lane 仅推进（用于导数/groupshared/barrier 的正确性）。
- 每次调用最多产出约 100 个状态；调用方需**反复调用直到返回空数组**。
- `PrepareInitial`（[:2013-2023](../renderdoc/driver/shaders/dxbc/dxbc_debug.cpp#L2013)）把每个初始变量作为 `{before=空, after=v}` 塞进第 0 步的 `changes`，从而定义了“初始寄存器状态”。

### 8.2 为什么能反向单步
`ShaderDebugState.changes` 里每个 `ShaderVariableChange` 同时含 `before`/`after`。UI 侧维护一份运行中的 `m_Variables`：
- 前进：对本步 changes 逐个 `var = after`（after 为空则删除该变量）；
- 后退：对本步 changes 逐个 `var = before`（before 为空则删除该变量）。

无需重新模拟即可任意方向移动。

---

## 9. 指令到源码/变量的映射

函数：`DXBCContainer::FillTraceLineInfo`（[dxbc_container.cpp:964-1033](../renderdoc/driver/shaders/dxbc/dxbc_container.cpp#L964)），在三个 Debug* 入口末尾调用。

```cpp
trace.instInfo.resize(m_DXBCByteCode->GetNumInstructions());       // :992
for (size_t i = 0; i < ...; i++) {
    trace.instInfo[i].instruction = (uint32_t)i;
    if (m_DebugInfo) m_DebugInfo->GetLineInfo(i, op.offset, trace.instInfo[i].lineInfo);  // 源码行/列
    if (op.line > 0)  trace.instInfo[i].lineInfo.disassemblyLine = extraLines + op.line;  // 反汇编行
    if (m_DebugInfo) m_DebugInfo->GetLocals(this, i, op.offset, trace.instInfo[i].sourceVars); // 局部变量映射
}
```

- `m_DebugInfo` 是从 DXBC 内嵌调试块解析出的 `IDebugInfo`（SDBG → [dxbc_sdbg.cpp](../renderdoc/driver/shaders/dxbc/dxbc_sdbg.cpp)，PDB/SPDB → [dxbc_spdb.cpp](../renderdoc/driver/shaders/dxbc/dxbc_spdb.cpp)）。
- `GetLineInfo` 给出**指令↔HLSL 源码行**，`GetLocals` 给出**该指令处的源码级局部变量**。这正是 UI 能“按源码行单步”“显示源码变量”的数据来源。

---

## 10. Replay Controller / Proxy 层

### 10.1 公共接口 `IReplayController`（[api/replay/renderdoc_replay.h](../renderdoc/api/replay/renderdoc_replay.h)）
- `DebugVertex`(:982)、`DebugPixel`(:1004)、`DebugThread`(:1014)、`DebugMeshThread`(:1025)、`ContinueDebug`(:1040)、`FreeTrace`(:1046)。
- `ContinueDebug` 契约（:1028-1039）：**每次至少执行一步**，返回若干步的列表；**返回空列表表示调试完成**。

### 10.2 实现 `ReplayController`（[replay/replay_controller.cpp](../renderdoc/replay/replay_controller.cpp)）
薄封装，转发到当前驱动 `m_pDevice`：
- `DebugPixel`(:1690-1705)、`DebugVertex`(:1672)、`DebugThread`(:1707)：调驱动后把 `ret->debugger` 记入 `m_Debuggers` 管理生命周期；
- `ContinueDebug`(:1743-1753)：**纯直通** `return m_pDevice->ContinueDebug(debugger);`；
- `FreeTrace`(:1755-1765)：`m_pDevice->FreeDebugger(...)` 后 `delete trace`。

对 D3D11，`D3D11Replay::ContinueDebug`（[d3d11_shaderdebug.cpp:2505-2518](../renderdoc/driver/d3d11/d3d11_shaderdebug.cpp#L2505)）把不透明句柄转回 `InterpretDebugger*`，在栈上建 `D3D11DebugAPIWrapper` 后调 `interpreter->ContinueDebug(&apiWrapper)`。

继承链：`ShaderDebugger`（不透明）← `DXBCContainerDebugger`（[dxbc_common.h:34](../renderdoc/driver/shaders/dxbc/dxbc_common.h#L34)）← `InterpretDebugger`（[dxbc_debug.h:251](../renderdoc/driver/shaders/dxbc/dxbc_debug.h#L251)）。

### 10.3 远程代理 `ReplayProxy`（[core/replay_proxy.cpp](../renderdoc/core/replay_proxy.cpp)）
跨机调试时序列化：
- `DebugPixel`（Proxied_ 在 :1657）：序列化 `eventId/x/y/inputs`，远端执行后 `SERIALISE_RETURN(*ret)` 把**整个 `ShaderDebugTrace` 回传一次**；
- `ContinueDebug`（Proxied_ 在 :1770-1796）：**把 debugger 指针当作 64 位标识值序列化**（从不跨线解引用），远端 `m_Remote->ContinueDebug(debugger)`，回传 `rdcarray<ShaderDebugState>` 批次；
- 净效果：大数据（trace）只传一次，之后客户端反复用不透明指针身份 ping `ContinueDebug` 取回状态批。

---

## 11. Qt UI：单步驱动与变量查看

文件：[qrenderdoc/Windows/ShaderViewer.cpp](../qrenderdoc/Windows/ShaderViewer.cpp) / [.h](../qrenderdoc/Windows/ShaderViewer.h)

### 11.1 关键成员（[ShaderViewer.h](../qrenderdoc/Windows/ShaderViewer.h)）
- `ShaderDebugTrace *m_Trace`(:332)、`rdcarray<ShaderDebugState> m_States`(:334)（完整时间线）、`size_t m_CurrentStateIdx`(:335)（游标）、`QList<ShaderVariable> m_Variables`(:336)（**当前步重建出的活变量**）、`m_VariablesChanged`(:339)（高亮）、`QSet<QPair<int,uint32_t>> m_Breakpoints`(:355)、`m_TempBreakpoint`(:356)。

### 11.2 累积状态（`debugShader`，[:1098-1218](../qrenderdoc/Windows/ShaderViewer.cpp#L1098)）
在 replay 线程循环调 `ContinueDebug` 直到空批：
```cpp
states->append(r->ContinueDebug(m_Trace->debugger));         // :1104
do {
    nextStates = r->ContinueDebug(m_Trace->debugger);        // :1118
    finished = nextStates.empty();
    states->append(std::move(nextStates));                   // :1127
} while (!finished && ...);
```
完成后回 GUI 线程 `m_States.swap(*states)`，用第 0 步 changes 播种 `m_Variables`，调 `updateDebugState()`。

### 11.3 正/反向回放（双向单步的实现）
- `applyForwardsChange()`（[:2936-3051](../qrenderdoc/Windows/ShaderViewer.cpp#L2936)）：先 `m_CurrentStateIdx++`，对本步每个 change：`after.name` 空→从 `m_Variables` 删除；否则 `*v = after` 或插入。
- `applyBackwardsChange()`（[:2844-2934](../qrenderdoc/Windows/ShaderViewer.cpp#L2844)）：镜像操作，`before.name` 空→删除（该变量本步诞生）；否则 `*v = before` 或插入；最后 `m_CurrentStateIdx--`。
- `step(bool forward, StepMode)`（[:2428-2609](../qrenderdoc/Windows/ShaderViewer.cpp#L2428)）：反汇编模式单次 apply；源码模式循环 apply 直到源码行变化，按 `StepInto/StepOver/StepOut` 比较 `callstack`，命中断点提前停。

### 11.4 运行/运行到光标/条件运行
- `runTo(targets, forward, condition)`（[:2719-2781](../qrenderdoc/Windows/ShaderViewer.cpp#L2719)）：循环 apply 直到到达目标指令 / 命中 `condition` 事件标志（如 `SampleLoadGather`、`GeneratedNanOrInf`）/ 命中断点行；
- `RunForward()`(:5838) = `runTo(~0U, true)`（跑到末尾）；
- `runToCursor(forward)`（:2611）：临时断点到当前指令再 `runTo`，随后恢复原断点集；
- 菜单/快捷键在 `debugShader` 内绑定（F5/Shift+F5、F10/F11 系列等）。

### 11.5 变量显示与源码映射（`updateDebugState`，[:4123-4659](../qrenderdoc/Windows/ShaderViewer.cpp#L4123)）
- **当前指令高亮**：用 `GetInstInfo(state.nextInstruction).lineInfo` 定位反汇编行与源码行，打 `CURRENT_MARKER`/`FINISHED_MARKER`；填 callstack 列表；
- **constants/inputs/resources 面板**：遍历 `m_Trace->sourceVars` + 未映射的常量块/输入/资源/采样器；
- **source variables 面板**：合并 trace 级 sourceVars 与**当前指令的局部 `GetCurrentInstInfo().sourceVars`**，跳过当前不存在的调试变量，按最近更新排序；
- **debug variables 面板**：直接来自活 `m_Variables`。

关键解析：`GetDebugVariable(DebugVariableReference)`（[:5573-5613](../qrenderdoc/Windows/ShaderViewer.cpp#L5573)）按 `type` 把源码变量引用解析到实际 `ShaderVariable*`：资源/采样器→`m_Trace` 对应表；`Input`→`m_Trace->inputs`；`Constant`→`m_Trace->constantBlocks`；**`Variable`→活 `m_Variables`**（这是从源码映射连到当前可变状态的关键一环）。

### 11.6 断点与高亮
- `ToggleBreakpointOnInstruction`（[:5659-5803](../qrenderdoc/Windows/ShaderViewer.cpp#L5659)）：由 `GetInstInfo(instruction).lineInfo` 同时算源码断点 `{fileIndex,line}` 与反汇编断点 `{-1,disasmLine}`，加/删 `BREAKPOINT_MARKER`；
- 断点在 `step`/`runTo`/`runToResourceAccess` 中通过 `m_Breakpoints.contains(...)` 消费。

---

## 12. 端到端数据流总结

```
用户在 UI 右键“Debug this pixel”
      │
      ▼
IReplayController::DebugPixel  (renderdoc_replay.h:1004)
      │  ReplayController::DebugPixel (replay_controller.cpp:1690)
      ▼
D3D11Replay::DebugPixel (d3d11_shaderdebug.cpp:1885)
   ├─ 重放 draw + 插桩 PS 抓取像素输入到 UAV，读回，挑选获胜 fragment
   ├─ InterpretDebugger::BeginDebug → 建 workgroup(4 lane)、填 inputs/constantBlocks
   └─ FillTraceLineInfo → 填 instInfo（源码行 + 局部变量）
      │ 返回 ShaderDebugTrace{ inputs, constantBlocks, ..., debugger=InterpretDebugger* }
      ▼
UI: ShaderViewer::debugShader  循环 ContinueDebug 直到空
      │  ReplayController::ContinueDebug (replay_controller.cpp:1743)
      │  D3D11Replay::ContinueDebug (d3d11_shaderdebug.cpp:2505)  建 D3D11DebugAPIWrapper
      ▼
InterpretDebugger::ContinueDebug (dxbc_debug.cpp:5586)
      └─ 每步 ThreadState::StepNext (dxbc_debug.cpp:2025)
            ├─ GetSrc 读操作数
            ├─ 采样/数学/资源 → DebugAPIWrapper 回到真实 GPU
            └─ SetDst 写结果 + 记录 before/after 差量 → ShaderDebugState.changes
      │ 返回 rdcarray<ShaderDebugState> （≤100/批）
      ▼
UI 累积 m_States；applyForwards/BackwardsChange 增量重建 m_Variables；
updateDebugState 刷新当前指令高亮 + 变量/源码/watch 面板
```

**核心洞见**：
1. **单步跟踪**=CPU 解释器逐条 `StepNext`，把“下一条指令 + 变量差量”打包成 `ShaderDebugState`。
2. **调试（前进/后退/运行到/断点）**=UI 在已生成的 `m_States` 时间线上，对 `ShaderVariableChange` 做正/反向增量应用，无需重算。
3. **变量查看**=活 `m_Variables`（底层寄存器/输入/常量）+ `SourceVariableMapping`/`instInfo` 还原为源码级变量。
4. **GPU 语义保真**=采样、超越函数、资源读取由 `DebugAPIWrapper` 回到真实 GPU 执行以保证位精确；像素输入靠**重放 + 插桩 PS**捕获。

---

## 13. 关键源码位置速查表

| 关注点 | 文件 | 函数 / 行 |
|--------|------|-----------|
| 公共数据模型 | `renderdoc/api/replay/shader_types.h` | `ShaderVariable`:330, `ShaderVariableChange`:901, `ShaderDebugState`:946, `ShaderDebugTrace`:1037 |
| 调试器公共 API | `renderdoc/api/replay/renderdoc_replay.h` | `DebugPixel`:1004, `ContinueDebug`:1040, `FreeTrace`:1046 |
| Replay 控制器实现 | `renderdoc/replay/replay_controller.cpp` | `DebugPixel`:1690, `ContinueDebug`:1743, `FreeTrace`:1755 |
| 远程代理 | `renderdoc/core/replay_proxy.cpp` | `Proxied_DebugPixel`:1657, `Proxied_ContinueDebug`:1770 |
| DX11 顶点调试入口 | `renderdoc/driver/d3d11/d3d11_shaderdebug.cpp` | `DebugVertex`:1520 |
| DX11 像素调试入口（重放+插桩） | 同上 | `DebugPixel`:1885 |
| DX11 compute 调试入口 | 同上 | `DebugThread`:2378 |
| DX11 ContinueDebug 分派 | 同上 | `ContinueDebug`:2505 |
| GPU 辅助 wrapper | 同上 | `D3D11DebugAPIWrapper`:46；`FetchSRV`:120, `FetchUAV`:194, `CalculateSampleGather`:1079, `CalculateMathIntrinsic`:1421 |
| 解释器类定义 | `renderdoc/driver/shaders/dxbc/dxbc_debug.h` | `GlobalState`:61, `DebugAPIWrapper`:145, `ThreadState`:178, `InterpretDebugger`:251 |
| 解释器建立会话 | `renderdoc/driver/shaders/dxbc/dxbc_debug.cpp` | `BeginDebug`:5065 |
| 单步主循环 | 同上 | `ContinueDebug`:5586 |
| 单条指令执行（大 switch） | 同上 | `StepNext`:2025 |
| 写目标 + 差量记录 | 同上 | `SetDst`:1155（before:1258 / after:1291） |
| 读源操作数 | 同上 | `GetSrc`:1441 |
| 控制流 | 同上 | `SWITCH`:4380, `LOOP/CONTINUE`:4457, `BREAK`:4501, `IF`:4537, `ELSE`:4581, `RET`:4625 |
| lane 活跃/收敛 | 同上 | `CalcActiveMask`:5452 |
| 导数 | 同上 | `DDX`:1385, `DDY`:1413 |
| 初始状态 | 同上 | `PrepareInitial`:2013 |
| 指令→源码/局部变量映射 | `renderdoc/driver/shaders/dxbc/dxbc_container.cpp` | `FillTraceLineInfo`:964 |
| 内嵌调试信息解析 | `renderdoc/driver/shaders/dxbc/dxbc_sdbg.cpp` / `dxbc_spdb.cpp` | `IDebugInfo::GetLineInfo` / `GetLocals` |
| UI 单步驱动 | `qrenderdoc/Windows/ShaderViewer.cpp` | `debugShader`:1098, `step`:2428, `runTo`:2719, `runToCursor`:2611 |
| UI 正/反向回放 | 同上 | `applyForwardsChange`:2936, `applyBackwardsChange`:2844 |
| UI 变量/源码面板刷新 | 同上 | `updateDebugState`:4123, `GetDebugVariable`:5573, `makeSourceVariableNode`:5123 |
| UI 断点 | 同上 | `ToggleBreakpointOnInstruction`:5659 |

---

## 附：与其他后端的关系

同一套 `ShaderDebugTrace`/`ShaderDebugState`/`ContinueDebug` 抽象被多后端复用：
- **DXBC(SM5)**：`DXBCDebug::InterpretDebugger`（本报告，D3D11 与 D3D12 共用）；
- **DXIL(SM6)**：`driver/shaders/dxil/dxil_debug.cpp`（`ContinueDebug`:10399）；
- **SPIR-V(Vulkan)**：`driver/shaders/spirv/spirv_debug_setup.cpp`（`ContinueDebug`:2650）。

D3D12 在 `driver/d3d12/d3d12_shaderdebug.cpp`（`ContinueDebug`:3934）按 shader 是 DXBC 还是 DXIL 分派到相应内核。因此本文分析的“解释器 + GPU 辅助 + 双向差量回放”模型，是 RenderDoc 所有着色器调试的通用范式，DX11 是其中最成熟、最典型的实现。
