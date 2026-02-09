# SOP 头脑风暴：智能排程系统

## 一、适用范围与技术架构

### 1. 流程范围
覆盖从**业务需求接收、资格审查、资源匹配（动物 + 房间）、排期锁定、冲突处理**，到**状态流转、自动更新**的全链路。

### 2. 技术实现
基于 **[Swarm.AI]** 的多 Agent 协作系统：
- 将业务逻辑封装为可调用的原子工具（Tools）。
- **Leader Agent** 负责管理全流程所有 Agent 的上下文，与客户/人工进行对话交互获取信息，同时监督整个流程的运转与状态实现。

### 3. 双端协同架构
1.  **移动端 (Mobile Port)**：需求采集与发起端（主要由 Sales 使用）。
2.  **PC 端 (PC Port)**：排程决策与管理端（角色不设限，由客户配置为 PM、SD 或其他角色协同）。

---

## 二、核心步骤与 Agent 职责

### Leader Agent
**定位**：承担全流程上下文管理与流程监督职责，负责意图识别、任务编排、多方案决策、人机交互。

#### 1. 全流程上下文管理
-   **统一状态**：维护所有 projects 的全生命周期上下文，同步各 Agent 的执行结果，确保各 Agent 基于统一的项目状态进行决策。
-   **历史追溯**：存储项目的所有历史数据，包括需求变更、资源匹配记录、排程方案、状态流转记录等。

#### 2. 双向交互与信息获取
-   **交互**：与客户、人工进行对话交互，接收非结构化需求、补充信息与反馈，同步更新项目上下文。
-   **追问**：针对需求缺失或逻辑错误，自动发起自然语言追问，确保需求信息完整。

#### 3. 状态流转驱动与提醒
-   **驱动资源状态流转**：`unlocked` → `soft_reservation` → `hard_lock` → `unlocked`，根据项目进度自动更新资源状态。
-   **定期提醒**：按预设周期（每周 / 两周 / 每月）向人工发送项目/供试品状态更新提醒，人工反馈后同步更新项目信息。
-   **超时处理**：每日扫描“未反馈”项目，根据“启动距离”梯度释放资源：
    *   启动 < 1 个月：逾期 **1 周** 未反馈 -> **自动释放**
    *   启动 1-3 个月：逾期 **2 周** 未反馈 -> **自动释放**
    *   启动 > 3 个月：逾期 **3 周** 未反馈 -> **自动释放**
-   **通知发送**：发送《资源锁定通知》、《超时释放资源通知》、《资源缺口统计表》、《资源锁使用复盘报告》等。

#### 4. 流程监督与异常处理
-   **监控**：检查各 Agent 的执行结果，确保流程按规则推进。
-   **异常响应**：
    *   **资源缺口**：当 ResourceManager 返回空列表或所有方案 score < 0.4，立即向人工发送资源缺口实时提醒（附带采购建议），创建待办任务并设置反馈时限。
    *   **超售异常**：若超售比例超过 **10%**，立即触发异常提醒，禁止生成超售方案，需人工调整。
    *   **失联处理**：若提醒连续 3 次发送失败，自动同步至项目管理部门负责人，触发人工介入核实方式。
    *   **状态异常**：若资源状态未按项目进度正常流转（如项目已启动但资源仍为 soft_reservation），立即发送异常提醒并锁定其他预约请求。
    *   **批量超时**：若单次扫描发现 ≥5 个超时限项目，触发批量超时预警。

#### 5. 定期复盘
-   按周/月生成《资源锁使用复盘报告》，推送至项目管理部门，统计超时释放、状态流转、反馈及时率等指标。

---

### 步骤 0：资源底数与分配初始化 (Data Agent)

**负责 Agent**：Data Agent
**定位**：底层数据基座，只读不写（写操作由排程锁定触发），保障“物理现状”的绝对真实。

#### 1. 业务目标
确保 Agent 的决策基于真实的实验室物理现状和现有分配。

#### 2. 执行逻辑
-   **全量底数加载**：同步实验室内所有动物库存（含供应商、给药史、洗脱期记录）和房间规格（GLP/非 GLP、容量）。
-   **实时占用同步**：标记资源物理状态为“检疫、存栏、实验中（给药/恢复期）、空置”，每日定时同步更新。
-   **分配快照建立**：记录存量项目的“硬锁 (Hard Lock)”与“软预订 (Soft Book)”，作为冲突检测的背景层（每日自动备份）。

#### 3. 恢复期房间动态腾挪与合并
1.  **触发**：dataAgent 检测到某项目进入“恢复期”状态。
2.  **执行**：
    *   Leader Agent 检索当前恢复期房间资源，判断是否可合并。
    *   生成《房间腾挪建议》发送至实验技术员。
    *   人工确认后，Leader Agent 更新房间状态（腾挪后房间标记为“空置”），并同步至全流程。

---

### 步骤 1：需求资格审查与信息补充 (Leader Agent)

**业务目标**：将销售的非结构化需求转化为完整、标准的表单，绑定全流程提醒与状态管理。

#### 1. 阶段划分
1.  **第一阶段：初期接触表格（快速建档）**
    *   *目的*：低门槛建档。销售初步沟通时填写核心“意向信息”，Leader Agent 建立临时会话 ID。
2.  **第二阶段：细化后完整表格（排程就绪）**
    *   *目的*：支持排程计算。通过自然语言追问补全的完整结构化数据，完成后可发起资源锁定。

#### 2. 推送触发逻辑
-   **完成度计算**：`(已填写的排程必填项数量 / 8) * 100%`。
-   **推送阈值**：**100%** 必填项有值。
-   **状态 A：待完善**（不可推送，按钮置灰） -> Leader Agent 持续追问。
-   **状态 B：已就绪**（允许推送，按钮高亮） -> 生成初步排期预览，创建定期提醒任务。
-   **合理性判定**：
    *   计划启动周不得早于当前周。
    *   逻辑矛盾校验（如：猴的体重与年龄明显不匹配）。

#### 3. 自然语言追问
若字段缺失或逻辑错误，Leader Agent 自动生成“提醒清单”并发起问询。

#### 4. 双向反馈机
初审阶段调用 ResourceManager 接口获取初步资源预存情况，反馈给用户以支持决策。

#### 5. 合同状态定级
-   有合同/中标 -> **P1-P4**（硬锁/确定性高）。
-   仅有业务机会 -> **P5**（执行超售策略）。

#### 6. 输出 Schema (Example)
```json
{
  "project_id": "MOCK-2026-001",
  "priority": "P2",
  "species": "猴",
  "count": 40,
  "project_status": "需求确认未锁定",
  "人工_info": {
    "人工_id": "人工008",
    "人工_name": "XXX",
    "contact": "XXX@xxx.com"
  },
  "constraints": {
    "supplier": ["供应商A", "供应商B"],
    "weight_range": [3.5, 5.0],
    "is_naive": true,
    "room_type": "猴房",
    "experiment_type": "长毒试验"
  },
  "expected_week": 12
}
```

---

### 步骤 2：多维资源匹配 (ResourceManager)

**负责 Agent**：ResourceManager
**业务目标**：执行动物与房间的精确匹配，并计算关键节点。

#### 2.1 动物资源匹配逻辑
1.  **深度筛选**：种属 -> 状态 -> 供应商 -> 年龄/体重 -> Naïve -> 给药史。
2.  **库存充足判定**：`实际匹配数 >= 需求数量 * (1 + 备用比例)`（默认 5%-10%）。
3.  **就位周算法**：
    *   **场景 A (自有 Naïve/新动物)**：`当前周 + max(0, 标准适应期 - 已在库静养时间)`。
    *   **场景 B (自有 Non-Naïve/洗脱动物)**：`上次实验结束周 + 标准洗脱期`。
    *   **场景 C (需外购)**：`当前周 + 采购及检疫周期`（通常 4-8 周）。
4.  **采购联动**：资源不足生成《月度资源缺口统计表》同步至 Leader Agent -> 采购岗。

#### 2.2 房间资源匹配逻辑
1.  **筛选**：类型（种属+实验类型）、容量。
2.  **状态排除**：排除“检疫期”和“实验中”的房间。
3.  **优先级**：空置期 > 存栏期（需确认可按时转出）。
4.  **可用周计算 (AvailableWeek Algorithm)**：
    *   **场景 A (空置)**：`当前周 + 消毒偏移量` (普通0.5周/检疫1周)。
    *   **场景 B (存栏)**：`存栏动物转出周 + 消毒偏移量`。
    *   *注*：`存栏动物转出周 = 当前动物就位周 - 清洗恢复期`。
5.  **启动周确定**：`项目实际启动周 = max(动物就位周, 房间可用周)`。

#### 2.3 线下人工复核
Leader Agent 生成《体检/采样任务单》 -> 人工（专题负责人） -> 录入体检合格结果 -> SimulationAgent。

---

### 步骤 3：排期发布与冲突排查 (SimulationAgent)

**负责 Agent**：SimulationAgent
**业务目标**：最终排期方案的博弈与锁定。

#### 3.1 冲突处理规则
1.  **检测范围**：周度重叠、优先级抢占、超售溢出。
2.  **优先级抢占**：
    *   高优先级（如 P1）可抢占低优先级（如 P3）的“软预订”。
    *   P1/P2 的“硬锁”**不可抢占**。
    *   执行阶段变更优先保障已启动项目。
3.  **超售规则**：
    *   仅适用于 P3-P5。
    *   比例 ≤ 10%。
    *   需签署《超售确认单》。

#### 3.2 资源锁定机制
-   **硬锁 (P1-P2)**：预定状态，占用实际物理容量，不可随意变更。
    *   *释放*：逾期 3 天未反馈自动释放。
-   **软预订 (P3-P5)**：预约状态，不占物理容量，可变性高。
    *   *释放*：逾期 5 天未反馈自动释放。

#### 3.3 方案预演
基于资源池生成 2-3 种方案（如：时效优先型、成本最低型、风险规避型）。
-   Leader Agent 驱动流转并推送方案对比表。

---

## 三、多 Agent 流程闭环

### 1. Agent 协作图谱与 Handoff 触发机制
系统采用 **[Swarm.AI]** 的动态交接机制，确保需求在不同专业智能体间无损流转。

| 发起 Agent | 接收 Agent | 触发条件 (Handoff Trigger) | 传递上下文 (Context) |
| :--- | :--- | :--- | :--- |
| **Leader Agent** | dataAgent | 需求结构化校验通过 (`is_valid=True`) | `RequirementSchema` (含优先级、种属、数量) |
| **dataAgent** | ResourceManager | 全量底数快照加载完成 | `InventorySnapshot` + `RequirementSchema` |
| **ResourceManager** | SimulationAgent | 发现 1 个及以上可用资源组合 | `MatchCandidates` (候选资源列表) |
| **SimulationAgent** | ConflictLock | 人工/人工 在对比方案后选定特定 Scenario | `SelectedScenario` + `LockIntensity` |
| **ConflictLock** | Leader Agent | 锁定成功或触发 P1 抢占逻辑 | `FinalSchedule` / `ConflictReport` |
| **Leader Agent** | ConflictLock | 定期扫描发现超时限未反馈项目 | `TimeoutProject` + `UnlockRequest` |
| **Leader Agent** | 人工（pc端侧） | 达到定期提醒节点 / 资源状态发生流转 / 超时释放触发 | `RemindContent` + `ResourceStatus` + `ProjectId` |
| **SystemTimer** | Leader Agent | 到达预设周期（每周/两周/每月）/ 每日定时扫描节点 | `CycleType` + `ScanRange` + `RemindTemplate` |

### 2. 核心算法伪代码实现

#### 2.1 综合可用周计算算法 (Available Week Algorithm)
```python
def calculate_available_week(req, room, animal_stock):  
    # 1. 动物就位周算法: 当前周 + 洗消/恢复期 (向上取整)  
    animal_lead_time = 1.0 if req.species in ["猴", "小鼠"] else 0.5  
    animal_ready_week = current_week + ceil(animal_lead_time)  
    
    # 2. 房间可用周算法  
    if room.status == "空置":  
        offset = 1.0 if room.type == "检疫房" else 0.5  
        room_ready_week = current_week + ceil(offset)  
    elif room.status == "存栏":  
        offset = 1.0 if room.type == "检疫房" else 0.5  
        room_ready_week = room.exit_week + ceil(offset)  
    else:  
        return float('inf') # 实验中，不可用  
          
    return max(animal_ready_week, room_ready_week, req.earliest_start_week)
```

#### 2.2 多方案损益评分模型 (P&L Scoring Model)
`SimulationAgent` 用于对不同排程策略进行量化评分。
```python
def score_scenario(scenario):  
    # 权重配置  
    W = {"time": 0.4, "utilization": 0.3, "risk": 0.3}  
    
    # 时效得分: 越早启动分越高  
    time_score = 1 / (scenario.start_week - current_week + 1)  
    
    # 资源利用率: 避免房间大材小用 (如 40 穴位房仅放 5 只动物)  
    util_score = scenario.actual_count / scenario.room_capacity  
    
    # 风险/损益损耗: 若挤占了潜在高价值(P1)项目的软预订，扣分  
    risk_score = 1 - (scenario.preempted_p1_p2_count * 0.5)  
    
    return (W["time"] * time_score) + (W["utilization"] * util_score) + (W["risk"] * risk_score)
```

#### 2.3 Leader Agent 核心算法
```python
def scan_timeout_projects(locked_resource_list, timeout_days=7, advance_days=30):  
    # 初始化返回结果
    unlock_projects = []
    current_date = datetime.now()
    for item in locked_resource_list:
        # 判定条件：启动前30天内 + 未反馈时长超7天
        days_before_start = (item.project_start_date - current_date).days
        no_feedback_days = (current_date - item.last_feedback_date).days
        if 0 < days_before_start <= advance_days and no_feedback_days >= timeout_days:
            # 标记为待释放项目
            unlock_projects.append(item.project_id)
            # 驱动资源状态从hard_lock/soft_reservation回退为unlocked
            update_resource_status(item.resource_id, target_status="unlocked")
            # 发送超时释放通知
            send_notification(item.人工_info.id, content=f"项目{item.project_id}已超时未反馈，资源已自动释放", attach_projects=[item.project_id])
    return unlock_projects

def drive_resource_status_flow(resource_id, project_status):  
    # 根据项目进度驱动资源状态流转：unlocked → soft_reservation → hard_lock
    current_status = get_resource_current_status(resource_id)
    if project_status == "需求确认未锁定" and current_status == "unlocked":
        update_resource_status(resource_id, target_status="soft_reservation")
    elif project_status == "人工选定方案" and current_status == "soft_reservation":
        update_resource_status(resource_id, target_status="hard_lock")
        # 发送资源锁定成功通知
        人工_info = get_project_人工_info(resource_id)
        send_notification(人工_info.id, content=f"资源{resource_id}已成功锁定，项目可按计划启动", attach_projects=[get_project_id_by_resource(resource_id)])
    elif project_status == "项目取消/超时" and current_status in ["soft_reservation", "hard_lock"]:
        update_resource_status(resource_id, target_status="unlocked")
    return get_resource_current_status(resource_id)

def send_cycle_remind(remind_cycle="weekly", 人工_info_list):  
    # 定期发送提醒，要求人工更新项目/供试品状态
    remind_content = {
        "weekly": "本周资源状态同步提醒：请更新负责项目的供试品准备进度、项目启动计划",
        "biweekly": "双周资源状态核查提醒：请确认项目资源锁定有效性，更新项目最新进展",
        "monthly": "月度项目进度复盘提醒：请提交项目资源使用情况、后续排期调整计划"
    }
    for 人工 in 人工_info_list:
        send_notification(人工.id, content=remind_content[remind_cycle], attach_projects=人工.manage_projects)
    return f"已向{len(人工_info_list)}位人工发送{remind_cycle}提醒"
```

### 3. 标准 API Schema 定义

#### 3.1 输入：需求契约 (RequirementSchema)
新增 `project_status` 字段，用于 Leader Agent 驱动资源状态流转；新增 `人工_info` 字段，关联提醒接收人信息。新增 `experiment_type` 字段，用于匹配房间类型和调整恢复期。
```json
{
  "project_id": "MOCK-2026-001",
  "priority": "P2",
  "species": "猴",
  "count": 40,
  "project_status": "需求确认未锁定",
  "人工_info": {
    "人工_id": "人工008",
    "人工_name": "XXX",
    "contact": "XXX@xxx.com"
  },
  "constraints": {
    "supplier": ["供应商A", "供应商B"],
    "weight_range": [3.5, 5.0],
    "is_naive": true,
    "room_type": "猴房",
    "experiment_type": "长毒试验"
  },
  "expected_week": 12
}
```

#### 3.2 输出：模拟方案对比 (SimulationResponse)
`resource_lock_status` 字段，展示当前资源状态，由 Leader Agent 同步更新。新增 `is_over_sold` 字段，标记是否为超售方案。
```json
{
  "scenarios": [
    {
      "type": "时效优先型",
      "score": 0.92,
      "start_week": 10,
      "room_id": "5402B",
      "p_and_l_impact": "正常收益",
      "risks": "无",
      "resource_lock_status": "soft_reservation",
      "is_over_sold": false
    },
    {
      "type": "成本最优型",
      "score": 0.85,
      "start_week": 12,
      "room_id": "3208B",
      "p_and_l_impact": "最大化空间利用率",
      "risks": "可能延迟2周启动",
      "resource_lock_status": "unlocked",
      "is_over_sold": false
    }
  ]
}
```

#### 3.3 扩展：Leader Agent 通知反馈 Schema
反馈 Schema，用于 人工 端接收提醒后反馈项目状态，完成 Agent 间上下文闭环，由 Leader Agent 接收并同步至所有相关 Agent。
```json
{
  "project_id": "MOCK-2026-001",
  "人工_feedback": {
    "update_time": "2026-XX-XX XX:XX:XX",
    "project_progress": "供试品准备中，预计W9到位",
    "resource_lock_need": true,
    "schedule_adjust": null
  },
  "resource_status_confirmation": "soft_reservation"
}
```

---

## 四、决策支持与异常处理 SOP

### 4.1 方案对比模板 (人工/人工 决策依据)

| 维度 | 方案 A (速度型) | 方案 B (稳健型) | 方案 C (低损型) |
| :--- | :--- | :--- | :--- |
| **启动周** | W10 (提前) | W12 (按需) | W14 (延后) |
| **资源利用率** | 60% (有浪费) | 95% (满载) | 80% |
| **潜在冲突** | 需抢占 P4 软预订 | 无冲突 | 无冲突 |
| **资源锁状态** | soft_reservation（人工 需 3 天内反馈） | soft_reservation（无反馈时限） | unlocked（备用） |
| **决策建议** | 适用于高优先级急单 | **推荐方案** | 备份方案 |

### 4.2 异常响应流程 (Human-in-the-loop)
Leader Agent 全流程参与，承接提醒、自动报告、超时处理动作：
1.  **缺口预警**：当 ResourceManager 返回空列表或所有方案 `score < 0.4`，Leader Agent 立即向人工发送资源缺口实时提醒，附带缺口详情和采购建议。
2.  **自动报告**：SimulationAgent 生成《资源缺口统计表》，Leader Agent 同步将报告推送至人工，同时创建待办任务并设置反馈时限。
3.  **人工接入**：Leader Agent 触发提醒后，实时监控人工/人工反馈状态；若超反馈时限未响应，再次发送催办提醒，并同步至销售部门。
4.  **超时释放**：若项目满足“启动前 1 个月内未反馈超 7 天”条件，Leader Agent 自动执行资源锁释放，并向相关方发送《超时释放资源通知》，同时更新资源状态至 `unlocked`。
5.  **定期复盘**：Leader Agent 按周/月生成《资源锁使用复盘报告》，推送至项目管理部门，统计超时释放、状态流转、反馈及时率等指标。

### 4.3 Leader Agent 专属异常处理
1.  **状态流转异常**：若资源状态未按项目进度正常流转（如项目已启动但资源仍为 soft_reservation、项目已取消但资源仍为 hard_lock），Leader Agent 立即发送异常提醒至人工，同时锁定该资源的其他预约请求，避免资源冲突；待状态核实修正后，再解锁该资源的预约权限。
2.  **批量超时**：若单次扫描发现 **≥5 个**超时限未反馈项目，Leader Agent 触发批量超时预警，向人工推送预警报告（含超时项目列表、超时时长、资源占用情况），要求人工核查，同时暂停这些项目的资源占用。
3.  **超售异常**：若 SimulationAgent 生成的排程方案中超售比例超过 **10%**，Leader Agent 立即触发异常提醒，禁止生成该超售方案，同时向人工发送超售预警，要求人工调整排期或协调资源，直至超售比例符合规则。
4.  **资源同步异常**：若 dataAgent 的资源快照与实际资源状态不一致（如房间状态未更新、动物库存数据错误），Leader Agent 在接收 dataAgent 快照时检测到数据异常，立即发送异常提醒至人工，要求核实并更新资源数据，同时暂停该项目的排程流程。

---

## 五、完整 Agent 协同工作流程实现
Leader Agent 在全流程中承担核心的交互、管控与上下文同步职责，各环节的具体执行逻辑如下：

1.  **需求捕获阶段**
    -   Leader Agent 通过 API 或对话渠道接收销售/客户的非结构化需求，解析为 JSON 对象；若信息不全，自动发起自然语言追问，补充缺失的必填项。
    -   需求校验通过后，Leader Agent 同步项目信息至 dataAgent，同时创建该项目的全生命周期提醒任务，绑定对应的人工和人工信息，设置定期提醒的周期。
    -   Leader Agent 会在方案生成后给到客户几个初步 option 方案，供其参考，并将用户的偏向反馈给人工衡量最终决策。

2.  **底数校验阶段**
    -   dataAgent 锁定当前物理世界的“真实状态”，排除已 `Hard_Lock` 的房间，生成资源底数快照后同步至 Leader Agent。
    -   Leader Agent 更新项目的资源上下文，确保后续所有 Agent 的决策基于最新的资源快照；若快照中存在资源数据异常，Leader Agent 立即触发资源同步异常处理流程。

3.  **匹配与计算阶段**
    -   ResourceManager 按“种属 -> 供应商 -> 状态”顺序执行硬性过滤，调用“就位周算法”计算各候选组合的最早可行周，输出候选结果后同步至 Leader Agent。
    -   Leader Agent 向人工发送资源候选结果提醒，附带筛选条件与可行周详情，同时记录人工的反馈时限；若资源不足，Leader Agent 接收 ResourceManager 生成的《月度资源缺口统计表》，推送给人工。

4.  **方案生成阶段**
    -   SimulationAgent 组合不同的资源路径，计算“损益评分”，产出 2-3 套差异化排程建议，生成方案后同步至 Leader Agent。
    -   Leader Agent 将生成的方案作为 option 反馈给第一阶段的客户，客户可进行选择，然后与人工进行协商。
    -   Leader Agent 驱动资源状态流转为 `soft_reservation`，同时向人工推送方案对比表，设置反馈时限；若方案存在超售异常，Leader Agent 触发超售异常处理流程，禁止该方案推送。

5.  **博弈锁定阶段**
    -   若人工选择方案，Leader Agent 自动触发 ConflictLock 机制检查是否有优先级抢占：P1 项目可直接将 P4 的 `Soft_Reservation` 状态重置为 `Unlocked`；锁定完成后，Leader Agent 驱动资源状态流转为 `hard_lock`，并向人工发送《资源锁定通知》，同时记录状态流转日志。
    -   若人工未选择方案/超反馈时限未响应，Leader Agent 实时监控状态，满足超时条件时自动释放资源锁，更新资源状态至 `unlocked`，并向所有相关 Agent 同步该状态，同时发送《超时释放资源通知》至人工。

6.  **闭环更新阶段**
    -   Leader Agent 更新 `room_experiment_combined.json` 快照，将最终排程结果 + 资源锁状态同步给人工并发送《资源锁定成功通知》，附带项目启动前的关键时间节点提醒。
    -   Leader Agent 按预设周期（每周/两周/每月）向人工发送项目/供试品状态更新提醒，人工反馈后，Leader Agent 同步更新项目信息至所有相关 Agent，确保全流程上下文一致。
    -   项目执行期间，Leader Agent 实时监控项目进度与资源状态匹配度，若出现项目延期/变更，自动触发资源锁状态复核，并根据新排期更新资源状态；若项目完成或取消，Leader Agent 驱动资源状态流转为 `unlocked`，完成资源释放。
