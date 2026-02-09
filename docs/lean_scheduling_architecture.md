# Intelligent Scheduling System: The "Lean & Mean" Architecture

## 1. Design Philosophy
> **TL;DR**: Stop trying to build Skynet to fix a pipe leak. Use the right tool for the job.

-   **Deterministic Core > Probabilistic Agents**: Scheduling is a mathematical constraint problem (Constraint Satisfaction Problem - CSP). It has a *best* answer, not a *creative* answer. Use solvers, not LLMs, for logic.
-   **LLM as Interface, Not Brain**: Use LLMs strictly for translation (Human -> JSON, JSON -> Human Explanation). Never trust an LLM to count to 10 reliably.
-   **Event-Driven State**: Use a robust State Machine (finite automata) and Message Queue, not "Agent Handoffs". Status changes should be transactional, not conversational.

---

## 2. System Architecture

The system is a standard microservice architecture with an "AI Gateway".

```mermaid
graph TD
    User[User (Mobile/PC)] --> Gateway[API Gateway]
    Gateway --> NLU[NLU Service (LLM Wrapper)]
    Gateway --> Scheduler[Scheduler Core (Solver)]
    Gateway --> BizData[Business Logic Service]
    
    NLU -->|Parses Natural Language| Gateway
    Scheduler -->|Queries Constraints| DB[(PostgreSQL)]
    Scheduler -->|Runs Optimization| Solver[OR-Tools / OptaPlanner]
    BizData -->|Calculates P&L| DB
```

### 3. Core Components

#### 3.1 NLU Service (The "Ear")
-   **Responsibility**: Single-mindedly parse user intent into a strictly typed JSON schema.
-   **Tech**: Fine-tuned small model (e.g., Llama-3-8B) or GPT-4o-mini with `response_format={type: "json_object"}`.
-   **Input**: "Need room for 40 monkeys, starting late March."
-   **Output**:
    ```json
    {
      "intent": "create_booking",
      "entities": {
        "species": "cynomolgus_monkey",
        "count": 40,
        "start_window": { "after": "2026-03-20", "before": "2026-03-31" }
      },
      "missing_fields": ["experiment_type"]
    }
    ```
-   **Constraint**: *Zero* business logic. Just extraction.

#### 3.2 Scheduling Engine (The "Brain")
-   **Responsibility**: Find valid time-space slots that satisfy hard constraints.
-   **Tech**: Google OR-Tools (CP-SAT) or a custom heuristic algorithm in Python/Go/Rust.
-   **Logic**:
    -   **Hard Constraints (Must Haves)**:
        -   Room Capacity >= Animal Count * 1.05
        -   Species Compatibility (No Rats next to Cats)
        -   Bio-safety Level (BSL-2 for certain viruses)
    -   **Soft Constraints (Nice to Haves)**:
        -   Minimize Gaps between projects (Utilization)
        -   Maximize Contiguous Blocks (Efficiency)
        -   Minimize Movement (Stability)
-   **Output**: Ranked list of valid `TimeSlot` objects.

#### 3.3 Business Logic Service (The "Wallet")
-   **Responsibility**: Calculate the P&L (Profit & Loss) impact of each valid slot.
-   **Logic**:
    -   **Opportunity Cost**: If a low-margin project blocks a high-value slot, penalize it.
    -   **Holding Cost**: Calculate cost of empty room days.
-   **Output**: A financial score for each option.

#### 3.4 Notification Service (The "Mouth")
-   **Responsibility**: Send deterministic alerts based on State Machine transitions.
-   **Channels**: Feishu/Lark, Email.
-   **Triggers**:
    -   `booking_created` -> Check Inventory -> Alert Procurement (if needed).
    -   `timeout_warning` -> Send "Confirm or Release" link.

---

## 4. Key Workflows

### W1: Intelligent Intake & Solving
1.  **User**: "Can I fit a P2 toxicology study for 30 dogs in mid-April?"
2.  **NLU**: Extracts `{ "species": "dog", "count": 30, "date": "2026-04-15", "type": "tox_p2" }`.
3.  **Scheduler**: Queries DB.
    -   *Constraint Check*: Dog rooms capacity.
    -   *Filter*: Exclude maintenance/quarantine.
    -   *Result*: Found 2 options.
        -   Option A: Room D-101 (Start April 12, Wait 3 days).
        -   Option B: Room D-105 (Start April 20, Perfect fit).
4.  **Business Logic**:
    -   Option A Score: 95 (Higher utilization).
    -   Option B Score: 80 (Leaves gap).
5.  **Gateway**: Returns standardized JSON response.
6.  **UI**: Renders comparison card.

### W2: Automated Optimization (The "Defrag")
*Run nightly via Cron Job.*
1.  **Scanner**: Identify all "Soft Reservations" (Low confidence).
2.  **Solver**: Simulate "What if we move these?"
3.  **Logic**: If moving Project X from Room A to Room B creates a massive contiguous block in Room A -> **Create Suggestion**.
4.  **Result**: Notification to Admin: "Optimization Opportunity: Move Project X to save 14 room-days."

## 5. Why This Beats the "Agent Swarm"
1.  **Predictability**: Input A always leads to Output B. Debugging is possible.
2.  **Performance**: Database indexes and mathematical solvers are order-of-magnitude faster than LLM round-trips.
3.  **Cost**: No per-token cost for the core logic layer.
4.  **Reliability**: "Hallucination" is impossible in a math solver. It either finds a solution or proves none exists.
