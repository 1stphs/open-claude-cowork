# Intelligent Scheduling Agent Skills - Orchestration Guide

## 1. System Overview
This directory contains the "Skills" (SOPs, Prompts, and Tools) for the **Intelligent Scheduling System**. The system is architected as a **Swarm.AI** multi-agent collaboration, where each agent has a distinct role and responsibility.

### Agent Roster
| Agent Name | Role | Responsibility | Skill Path |
| :--- | :--- | :--- | :--- |
| **Leader Agent** | Orchestrator & Interface | User interaction, state management, process supervision, exception handling. | [Leader Agent Skill](./leader_agent/SKILL.md) |
| **Data Agent** | Truth Keeper | Maintaining the "Physical Reality" of lab resources (Inventory, Rooms). Read-Only basis for decisions. | [Data Agent Skill](./data_agent/SKILL.md) |
| **ResourceManager** | Matcher & Filter | Executing hard constraints logic to find valid Resource/Room candidates. | [Resource Manager Skill](./resource_manager/SKILL.md) |
| **SimulationAgent** | Strategist | Generating scheduling scenarios, calculating P&L scores, and managing conflict resolution/locking. | [Simulation Agent Skill](./simulation_agent/SKILL.md) |

---

## 2. Agent Collaboration & Handoffs

The system uses a dynamic handoff mechanism. Agents transfer control based on specific triggers and pass necessary context.

| Initiator | Receiver | Trigger Condition | Context Transferred |
| :--- | :--- | :--- | :--- |
| **Leader Agent** | Data Agent | Requirement is structured & valid (`is_valid=True`) | `RequirementSchema` |
| **Data Agent** | Resource Manager | Inventory/Room snapshots are generated | `InventorySnapshot` + `RequirementSchema` |
| **Resource Manager** | Simulation Agent | ≥1 valid candidate resource found | `MatchCandidates` |
| **Simulation Agent** | Leader Agent | Scenarios generated and ready for user selection | `ScenarioOptions` |
| **Leader Agent** | Simulation Agent | User selects a specific scenario | `SelectedScenario` + `LockIntensity` |
| **Simulation Agent** | Leader Agent | Locking confirmed (Success/Fail) | `FinalSchedule` / `ConflictReport` |

---

## 3. Global Schemas

### Project Information (RequirementSchema)
Shared across all agents to maintain context.
```json
{
  "project_id": "STRING",
  "priority": "P1-P5",
  "species": "STRING",
  "count": "INTEGER",
  "project_status": "STRING", // driven by Leader Agent
  "human_info": {
    "id": "STRING",
    "name": "STRING",
    "contact": "STRING"
  },
  "constraints": {
    "supplier": ["ARRAY"],
    "weight_range": [MIN, MAX],
    "is_naive": "BOOLEAN",
    "room_type": "STRING",
    "experiment_type": "STRING"
  },
  "expected_week": "INTEGER"
}
```

### Resource Lock Status
Tracks the lifecycle of a resource reservation.
- **unlocked**: Available for any use.
- **soft_reservation**: Tentatively booked (P3-P5), expires if not confirmed.
- **hard_lock**: Confirmed booking (P1-P2), occupies physical capacity.

---

## 4. Usage Guide
To use these skills:
1.  **Start with the Leader Agent**: Initialize the interaction using the Leader Agent's system prompt.
2.  **Mount Tools**: Ensure the Python tools defined in each skill are available to the respective agent.
3.  **Monitor State**: Use the Leader Agent's state management logic to track where a project is in the pipeline.
