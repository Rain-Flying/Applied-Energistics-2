# AE2 Tick 系统技术文档

## 概述

AE2 通过 [TickHandler](file:///workspace/src/main/java/appeng/hooks/ticking/TickHandler.java) 单一入口注册了 NeoForge 的 `ServerTickEvent` 和 `LevelTickEvent` 事件监听器，在 ServerTick 和 LevelTick 的开始/结束四个时机挂载处理逻辑。Tick 事件遵循 **ServerTick → LevelTick** 的嵌套关系：一次 ServerTick 内包含多个 LevelTick（每个已加载的维度各一次）。

---

## 1. 事件注册

[TickHandler.init()](file:///workspace/src/main/java/appeng/hooks/ticking/TickHandler.java#L96-L104) 在 `NeoForge.EVENT_BUS` 上注册了 5 个监听器：

```java
NeoForge.EVENT_BUS.addListener(this::onServerTickStart);      // ServerTickEvent.Pre
NeoForge.EVENT_BUS.addListener(this::onServerTickEnd);        // ServerTickEvent.Post
NeoForge.EVENT_BUS.addListener(this::onServerLevelTickStart); // LevelTickEvent.Pre
NeoForge.EVENT_BUS.addListener(this::onServerLevelTickEnd);   // LevelTickEvent.Post
NeoForge.EVENT_BUS.addListener(this::onUnloadChunk);          // ChunkEvent.Unload
NeoForge.EVENT_BUS.addListener(EventPriority.LOWEST, this::onUnloadLevel); // LevelEvent.Unload
```

---

## 2. 四个 Tick 时机详解

### 2.1 ServerTick.Start — `onServerTickStart`

**触发时机**：每个服务器 tick 的开始（在任何 Level tick 之前）。

**执行逻辑**（[TickHandler.onServerTickStart](file:///workspace/src/main/java/appeng/hooks/ticking/TickHandler.java#L278-L294)）：

1. **重置计时器**：清零 `processQueueElementsProcessed` 和 `processQueueElementsRemaining` 计数器，重置 `stopWatch` 秒表（用于后续队列处理的超时监控）。

2. **遍历所有 Grid，调用 `grid.onServerStartTick()`**：对 `ServerGridRepo` 中所有已注册的 Grid 依次调用 `onServerStartTick()`。

**Grid 层分发**（[Grid.onServerStartTick](file:///workspace/src/main/java/appeng/me/Grid.java#L218-L226)）：

Grid 遍历 `serverStartTickServices` 数组，调用其中每个服务的 `onServerStartTick()`。该数组在 [GridServices.createServices](file:///workspace/src/main/java/appeng/api/networking/GridServices.java#L99-L130) 中通过反射判断哪些服务 override 了该方法来动态构建。

**实际挂载的服务**：

| 服务 | 文件 | 功能 |
|------|------|------|
| **TickManagerService** | [TickManagerService.java](file:///workspace/src/main/java/appeng/me/service/TickManagerService.java#L69-L71) | 递增 `currentTick` 计数器，作为网格内部 tick 计数的时钟源 |
| **EnergyService** | [EnergyService.java](file:///workspace/src/main/java/appeng/me/service/EnergyService.java#L188-L206) | 处理被动发电器（Passive Generator）：遍历所有被动发电器，选出速率最高者作为当前 overlay grid 的发电器，抑制其他速率较低的发电器 |

---

### 2.2 LevelTick.Start — `onServerLevelTickStart`

**触发时机**：每个 ServerLevel 的 tick 开始（紧跟 ServerTick.Start 之后，每个维度一次）。

**执行逻辑**（[TickHandler.onServerLevelTickStart](file:///workspace/src/main/java/appeng/hooks/ticking/TickHandler.java#L232-L256)）：

1. **处理 Level 级回调队列**：从 `callQueue` 中取出该 Level 的 `Queue<ILevelRunnable>`，调用 `processQueue()` 逐个执行，有 25ms 超时限制（与全局队列共享同一个 stopwatch），超时后剩余回调推迟到下一 tick。处理完后将队列放回 `callQueue`，并合并可能在此期间新添加的任务。

2. **同步 Grid 增删**：调用 `grids.updateNetworks()` 将积压的 `toAdd`/`toRemove` 集合同步到正式的网络集合中。

3. **遍历所有 Grid，调用 `grid.onLevelStartTick(level)`**。

**Grid 层分发**（[Grid.onLevelStartTick](file:///workspace/src/main/java/appeng/me/Grid.java#L228-L236)）：

Grid 遍历 `levelStartTickServices` 数组，调用每个服务的 `onLevelStartTick(level)`。

**实际挂载的服务**：当前（截至此版本）没有任何服务 override `onLevelStartTick`，所有服务使用接口默认空实现。该钩子为扩展预留。

---

### 2.3 LevelTick.End — `onServerLevelTickEnd`

**触发时机**：每个 ServerLevel 的 tick 结束（该维度 tick 逻辑执行完毕后）。

**执行逻辑**（[TickHandler.onServerLevelTickEnd](file:///workspace/src/main/java/appeng/hooks/ticking/TickHandler.java#L258-L276)）：

1. **模拟合成任务**：调用 `simulateCraftingJobs(level)`，从 `craftingJobs` 中取出该 Level 的合成计算任务集合，按每个任务分配的时间片（`AEConfig.craftingCalculationTimePerTick` / 任务数）依次推进模拟，已完成的任务从集合中移除。

2. **初始化待就绪的 BlockEntity**：调用 `readyBlockEntities(level)`，遍历该 Level 的 `ServerBlockEntityRepo` 中待初始化的 BlockEntity，按 chunk 逐一检查 `areBlockEntitiesTicking`（即 chunk 是否已加载且 ticking），然后调用其 `initFunction`（即 `GridHelper.onFirstTick`）完成初始化。

3. **遍历所有 Grid，调用 `grid.onLevelEndTick(level)`**。

**Grid 层分发**（[Grid.onLevelEndTick](file:///workspace/src/main/java/appeng/me/Grid.java#L238-L246)）：

Grid 遍历 `levelEndTickServices` 数组，调用每个服务的 `onLevelEndTick(level)`。

**实际挂载的服务**：

| 服务 | 文件 | 功能 |
|------|------|------|
| **TickManagerService** | [TickManagerService.java](file:///workspace/src/main/java/appeng/me/service/TickManagerService.java#L74-L76) | 处理该 Level 的网格节点 tick 队列：从 `upcomingTicks` 中取出该 Level 的 `PriorityQueue<TickTracker>`，按 `nextTick` 时间依次驱动到期节点，根据返回值 `TickRateModulation` 调整 tick 频率（URGENT/FASTER/SLOWER/SAME/SLEEP） |

---

### 2.4 ServerTick.End — `onServerTickEnd`

**触发时机**：每个服务器 tick 的结束（所有 Level tick 完成之后）。

**执行逻辑**（[TickHandler.onServerTickEnd](file:///workspace/src/main/java/appeng/hooks/ticking/TickHandler.java#L296-L318)）：

1. **遍历所有 Grid，调用 `grid.onServerEndTick()`**。

2. **处理全局回调队列**：调用 `processQueue(this.serverQueue, null)` 处理跨 Level 的全局回调（`level == null` 的回调），同样有 25ms 超时限制。

3. **超时检查**：如果 stopwatch 累计超过 25ms，输出警告日志。

4. **递增 tickCounter**：`tickCounter++`。

**Grid 层分发**（[Grid.onServerEndTick](file:///workspace/src/main/java/appeng/me/Grid.java#L248-L256)）：

Grid 遍历 `serverEndTickServices` 数组，调用每个服务的 `onServerEndTick()`。

**实际挂载的服务**：

| 服务 | 文件 | 功能 |
|------|------|------|
| **TickManagerService** | [TickManagerService.java](file:///workspace/src/main/java/appeng/me/service/TickManagerService.java#L79-L81) | 处理 `null` Level 的节点 tick 队列（虚拟节点 / 无世界归属的节点） |
| **EnergyService** | [EnergyService.java](file:///workspace/src/main/java/appeng/me/service/EnergyService.java#L209-L269) | (1) 注入被动发电器能量；(2) 检查能量阈值变化并通知 EnergyWatcher；(3) 更新滑动平均能耗/注入统计；(4) 从 providers 提取能量维持网络运行，判断是否有电；(5) 缓冲 30 tick 后更新 `publicHasPower` 状态并发布 `GridPowerStatusChange` 事件 |
| **PathingService** | [PathingService.java](file:///workspace/src/main/java/appeng/me/service/PathingService.java#L92-L143) | (1) 重新计算控制器状态；(2) 执行频道重分配（repath/reboot）：根据有无控制器分别走 AdHoc 或 PathingCalculation 路径分配频道；(3) 触发成就检测；(4) 通知所有节点频道更新完成 |
| **StorageService** | [StorageService.java](file:///workspace/src/main/java/appeng/me/service/StorageService.java#L100-L108) | 更新缓存物品列表（`cachedAvailableStacks`），检测物品数量变化并通知所有 `IStorageWatcherNode` 观察者 |
| **CraftingService** | [CraftingService.java](file:///workspace/src/main/java/appeng/me/service/CraftingService.java#L141-L215) | (1) 更新 CPU 集群列表；(2) 清理死亡合成链接；(3) 驱动每个 CPU 的合成逻辑 tick；(4) 检测合成中/可合成物品变化，通知 `ICraftingWatcherNode` 观察者 |

---

## 3. 核心数据结构

### 3.1 TickHandler 内部结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `serverQueue` | `Queue<ILevelRunnable>` | 全局（跨 Level）回调队列 |
| `callQueue` | `Map<LevelAccessor, Queue<ILevelRunnable>>` | 按 Level 分的回调队列 |
| `craftingJobs` | `Multimap<LevelAccessor, CraftingCalculation>` | 按 Level 分的合成模拟任务 |
| `blockEntities` | `ServerBlockEntityRepo` | 待初始化的 BlockEntity 仓库（按 Level → Chunk 组织） |
| `grids` | `ServerGridRepo` | 活跃 Grid 仓库（含增删缓冲区） |
| `stopWatch` | `Stopwatch` | 跨队列的累计计时器，用于 25ms 超时控制 |
| `tickCounter` | `long` | 服务器 tick 计数 |

### 3.2 ServerGridRepo

[ServerGridRepo](file:///workspace/src/main/java/appeng/hooks/ticking/ServerGridRepo.java) 维护三组集合：
- `networks`：当前活跃的 Grid
- `toAdd`：待添加的 Grid（积压到 `updateNetworks()` 时处理）
- `toRemove`：待移除的 Grid（积压到 `updateNetworks()` 时处理）

`updateNetworks()` 在 LevelTick.Start 时被调用，先处理移除再处理添加。

### 3.3 ServerBlockEntityRepo

[ServerBlockEntityRepo](file:///workspace/src/main/java/appeng/hooks/ticking/ServerBlockEntityRepo.java) 按 `LevelAccessor → ChunkPos → List<FirstTickInfo>` 三层结构存储待初始化的 BlockEntity。在 LevelTick.End 时，对已 ticking 的 chunk 批量执行初始化回调。

### 3.4 TickManagerService 内部结构

[TickManagerService](file:///workspace/src/main/java/appeng/me/service/TickManagerService.java) 维护三类节点集合：

| 集合 | 类型 | 说明 |
|------|------|------|
| `alertable` | `Map<IGridNode, TickTracker>` | 可被唤醒的节点（允许外部 `alertDevice` 立即排程） |
| `sleeping` | `Map<IGridNode, TickTracker>` | 休眠节点（不参与 tick，直到被 `wakeDevice` 唤醒） |
| `awake` | `Map<IGridNode, TickTracker>` | 活跃节点（在 tick 队列中） |
| `upcomingTicks` | `Map<Level, PriorityQueue<TickTracker>>` | 按 Level 分的优先级队列（按 `nextTick` 排序） |

Tick 频率根据节点返回的 `TickRateModulation` 动态调整：
- `URGENT` → 设为 `minTickRate`
- `FASTER` → `currentRate - 2`
- `SLOWER` → `currentRate + 1`
- `IDLE` / `SLEEP` → 设为 `maxTickRate`（休眠）
- `SAME` → 保持 `currentRate`

---

## 4. 执行时序图

```
ServerTick.Start
  ├── TickHandler: 重置计时器
  ├── Grid[*].onServerStartTick()
  │     ├── TickManagerService: currentTick++
  │     └── EnergyService: 选择最优被动发电器
  │
  ├── LevelTick.Start (维度 A)
  │     ├── TickHandler: 处理 Level A 回调队列 (25ms 限制)
  │     ├── ServerGridRepo.updateNetworks() [同步 Grid 增删]
  │     └── Grid[*].onLevelStartTick(A) [当前无服务实现]
  │
  ├── LevelTick.End (维度 A)
  │     ├── TickHandler: 模拟合成任务 (simulateCraftingJobs)
  │     ├── TickHandler: 初始化 BlockEntity (readyBlockEntities)
  │     └── Grid[*].onLevelEndTick(A)
  │           └── TickManagerService: 处理 Level A 的节点 tick 队列
  │
  ├── ... (其他维度重复 LevelTick.Start/End) ...
  │
  └── ServerTick.End
        ├── Grid[*].onServerEndTick()
        │     ├── TickManagerService: 处理 null Level 的节点 tick 队列
        │     ├── EnergyService: 注入被动能量、更新通电状态
        │     ├── PathingService: 频道重分配
        │     ├── StorageService: 更新缓存物品列表
        │     └── CraftingService: 驱动 CPU 合成逻辑
        ├── TickHandler: 处理全局回调队列 (25ms 限制)
        └── TickHandler: tickCounter++
```

---

## 5. 回调队列机制

TickHandler 提供 `addCallable(LevelAccessor level, ILevelRunnable c)` 方法供外部模块注册延迟回调：

- `level == null`：注册到全局 `serverQueue`，在 **ServerTick.End** 时执行
- `level != null`：注册到对应 Level 的队列，在下一帧的 **LevelTick.Start** 时执行

两个队列共享同一个 25ms 的 `stopwatch` 超时限制，未完成的回调推迟到下一帧继续处理。

---

## 6. 关键设计要点

1. **Grid 增删延迟同步**：Grid 的创建和销毁不在事件发生时立即生效，而是通过 `ServerGridRepo` 的 `toAdd`/`toRemove` 缓冲区积压，在 **LevelTick.Start** 时统一处理，避免在遍历过程中修改集合。

2. **BlockEntity 延迟初始化**：AE2 的 BlockEntity 不在 `setBlockState` 或 chunk 加载时立即初始化，而是等到 **LevelTick.End** 时 chunk 确认处于 ticking 状态后才初始化，避免在未加载完成的 chunk 中访问世界状态。

3. **Tick 频率自适应**：网格节点的 tick 频率不是固定的，而是根据节点返回的 `TickRateModulation` 动态调整，范围由 `TickingRequest.minTickRate()` 和 `maxTickRate()` 限定。

4. **被动发电器 overlay grid**：EnergyService 通过 overlay grid 机制将多个通过石英纤维连接的网络视为一个能量网络，在 ServerTick.Start 选出最优发电器，在 ServerTick.End 统一注入能量。

5. **25ms 超时保护**：所有回调队列处理都有 25ms 硬限制（相对于 50ms 的理想 tick 时间），防止 AE2 逻辑占用过多 tick 时间导致服务器卡顿。

6. **服务发现机制**：`GridServices.createServices()` 在注册时通过反射检查每个服务类是否 override 了四个 tick 方法（与 `IGridServiceProvider` 接口的默认方法比较 declaring class），自动将服务归入对应的 tick 阶段数组，避免运行时反射开销。