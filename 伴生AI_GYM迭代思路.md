# 伴生AI GYM 迭代思路

**整理人：阿康（康帆）**
**日期：2026-04-07**
**依据：代码 + 伴生AI使用Wiki + 现有 gym 任务文件**

---

## 一、现状梳理

### 已有 gym 任务文件

| 文件 | 类型 | 说明 |
|------|------|------|
| `23301246_npcai_gym.tsp` | 任务Spoon | **当前唯一的 npcai_gym 主任务**，已包含跟随AI和载具相关节点 |
| `23301225_taskgym.tsp` / `Vehicle_gym.tsp` | 任务Spoon | 任务类/载具类 gym |
| `gym_test.py` | 自动化脚本 | 当前自动化框架已有 `agent_gym` 的 linqin_tag 映射（`"agent_gym":"npc"`） |

### 当前 npcai_gym.tsp 已覆盖节点

从文件内容中可读出，现有任务已测试了：

- `StartTaskBTNpcTraceUnit2`（跟随AI，2处使用，对应两个 AgentUnit）
- `GeneralTeleport`（传送）
- `OnEnterOrExitVehicle`（上下车事件监听）
- `LeaveVehicle`（下车动作）
- `OnVehicleEnterArea`（载具进入区域）
- 路点组：带着NPC前行（带着班茜往前跑1/2/3）

### 尚未覆盖的核心节点

对比 Wiki 和全部 NPC 节点代码，以下节点**尚未在 npcai_gym 中被测试**：

| 节点 | 类名 | 重要程度 |
|------|------|---------|
| 路径带路AI | `StartBTCheckPointMove2` | ⭐⭐⭐⭐⭐ |
| 单点移动AI | `StartTaskBTNpcMove2` | ⭐⭐⭐⭐⭐ |
| NPC转向 | `NpcChangeDirection` | ⭐⭐⭐ |
| 动作播放 | `StartTaskBTNpcAction` | ⭐⭐⭐⭐ |
| 取消行为树 | `CancelNpcBehTree2` | ⭐⭐⭐ |
| NPC上车（主动） | `TaskBTNpcToVehicle` | ⭐⭐⭐⭐ |
| 自由伴随 | `StartFreedomCompanion` | ⭐⭐⭐ |

---

## 二、GYM 迭代思路

### 整体原则

> 以"**最小可复现单元**"为原则，每个子任务只测试一个核心节点的核心功能，保证：
> 1. 失败时定位快速、准确
> 2. 新增节点只需追加子任务，互不干扰
> 3. 与 `gym_test.py` 中已有的 `agent_gym` case 衔接

---

### Phase 1：核心移动节点（优先级最高）

#### 子任务1：跟随AI基础验证（已有，补充用例）

- **节点**：`StartTaskBTNpcTraceUnit2`
- **检查点**：
  - NPC 是否正确跟随玩家移动（`arriveDistance` 范围内停止）
  - `enableEndCondition=true` 时，超出 `maxRange` 是否触发 False 输出，低于 `minRange` 是否触发 True 输出
  - `isFollowToVehicle=true` 时，玩家上载具后 NPC 是否也跟随
  - 移动类型（`traceType`）：动态变速 vs 匹配玩家速度 的差异表现

#### 子任务2：带路AI（路径移动）🆕

- **节点**：`StartBTCheckPointMove2`
- **参数来源**：`Client/Assets/Script/Spoon/Actions/ClientActions/Npc/StartBTCheckPointMove2.cs`
- **检查点**：
  - `isLeadingWay=true`：NPC 带路时，玩家落后是否触发等待和催促对话（`waitingDialogId`）
  - `isLeadingWay=false`（定点移动）：NPC 直接移动到终点，无等待行为
  - 路点上的行为功能：动作播放/步态切换是否在对应路点正常触发
  - 多方向停步（`leadingBreakTurn`）：NPC 等待时是否朝向玩家
  - `dontLimitBasicMove=false`：玩家速度是否被限制在 NPC 步速内

#### 子任务3：单点移动AI 🆕

- **节点**：`StartTaskBTNpcMove2`
- **参数来源**：`Client/Assets/Script/Spoon/Actions/ClientActions/Npc/StartTaskBTNpcMove2.cs`
- **检查点**：
  - `IsWayPointMove=true`：NPC 沿路点组移动到终点
  - `IsWayPointMove=false` + `isUsePoint=true`：NPC 移动到指定坐标（走 navmesh 寻路）
  - `IsLoop=true`：NPC 是否循环移动
  - 到达终点后后续节点是否正常触发（Done 端口）

---

### Phase 2：表演编排节点

#### 子任务4：NPC 动作播放 🆕

- **节点**：`StartTaskBTNpcAction`
- **检查点**：
  - NPC 是否播放指定动作并在完成后继续后续节点
  - 动作队列多个动作依次播放是否正常

#### 子任务5：NPC 转向 🆕

- **节点**：`NpcChangeDirection`
- **检查点**：
  - 角度差 ≥10° 时，NPC 是否正确转向（注意：旧节点默认角度可能不是10，已有 wiki 说明需手动改为10）
  - 转向后是否停止踱步（避免反复抖动的已知问题）

#### 子任务6：取消行为树 🆕

- **节点**：`CancelNpcBehTree2`
- **检查点**：
  - 行为树运行中调用该节点，AI 是否立刻停止当前行为
  - 取消后 NPC 状态是否恢复 idle

---

### Phase 3：进阶场景

#### 子任务7：无缝衔接 Timeline 🆕

- **相关节点**：移动节点 + `JumpTimeline` / `PlayClientTimeline` 并行，勾选"无缝衔接TL"
- **前置要求**：角色移动动作已在「策划/动作编辑/脚基运动分析曲线」注册（StrideRemapping）
- **检查点**：
  - NPC 以 idle/walk/run 姿态无缝进入 TL
  - 终点位置/朝向与 TL 初始位置/旋转一致时，过渡是否自然
  - 移动路径是否预留了3步以上距离（衔接脚设置为左脚/右脚的移动姿态衔接）

#### 子任务8：NPC 开门 🆕

- **方式**：NPC 打上 Tag `UnitGamePlayTrait_PushRigidBodyDoor`
- **检查点**：
  - NPC 携带 Tag 时，移动路径穿过门是否自动触发开门
  - 过门后 Tag 是否被正确清除（性能优化）

---

## 三、与自动化框架的衔接建议

当前 `gym_test.py` 已支持 `agent_gym` case 的自动化运行和上报：

```python
# gym_test.py 中已有映射
"agent_gym": "npc"   # 对应林檎 tag "npc"
```

**建议方案**：

1. **保持现有 Alias 不变**：`23301246_npcai_gym` 对应 `agent_gym` case，继续复用，在其中追加子任务
2. **如节点间互相干扰**：可拆分为多个独立 tsp（如 `npcai_gym_trace`、`npcai_gym_lead` 等），并在 QA 平台分别注册 case，便于定向排查
3. **失败通知**：框架已支持自动开易协作工单 + POPO 消息推送，新 case 注册后即可复用

---

## 四、施工优先级建议

| 优先级 | 节点 | 理由 |
|--------|------|------|
| P0 | `StartBTCheckPointMove2`（带路） | 使用最广泛，迭代最频繁（见 Wiki 备注） |
| P0 | `StartTaskBTNpcMove2`（单点移动） | 正在迭代中（见工单 #825796） |
| P1 | `StartTaskBTNpcTraceUnit2` 补充用例 | 已有但覆盖不全 |
| P1 | `NpcChangeDirection`（转向） | 有已知 bug（踱步问题），需回归验证 |
| P2 | `StartTaskBTNpcAction`（动作） | 表演类，相对独立 |
| P2 | 无缝衔接 Timeline | 新功能（2026.1.26），需专项验证 |
| P3 | `CancelNpcBehTree2` / NPC开门 | 较少使用，放后期 |

---

## 五、相关代码路径参考

| 节点 | 代码路径 |
|------|---------|
| `StartTaskBTNpcTraceUnit2` | `Client/Assets/Script/Spoon/Condition/ClientConditions/Npc/StartTaskBTNpcTraceUnit2.cs` |
| `StartTaskBTNpcMove2` | `Client/Assets/Script/Spoon/Actions/ClientActions/Npc/StartTaskBTNpcMove2.cs` |
| `StartBTCheckPointMove2` | `Client/Assets/Script/Spoon/Actions/ClientActions/Npc/StartBTCheckPointMove2.cs` |
| `NpcChangeDirection` | `Client/Assets/Script/Spoon/Actions/ClientActions/Npc/NpcChangeDirection.cs` |
| `StartTaskBTNpcAction` | `Client/Assets/Script/Spoon/Actions/ClientActions/Npc/StartTaskBTNpcAction.cs` |
| `CancelNpcBehTree2` | `Client/Assets/Script/Spoon/Actions/ClientActions/Npc/CancelNpcBehTree2.cs` |
| 现有 gym 任务 | `bin/ServerRes/SpoonGraph/Task/23301246_npcai_gym.tsp` |
| 自动化脚本 | `CheckEnv/Tool/Script/AutoQA/QACI/gym_test.py` |
