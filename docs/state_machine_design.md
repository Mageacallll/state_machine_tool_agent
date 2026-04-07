# State-Machine-Augmented Tool Calling for τ²-Bench Retail

## 1. Motivation

LLM agents in τ²-Bench often fail not because they lack raw language ability, but because they misuse tools:

- Calling the wrong write tool for the current order status
- Skipping confirmation before mutation
- Performing actions in the wrong sequence
- Getting distracted by too much raw database output
- Making unrecoverable mistakes that still look superficially reasonable

**Goal:**  
Introduce a State Machine (SM) layer to constrain and guide tool usage so that agents follow valid business workflows more reliably, especially for medium/lightweight models.

---

## 2. Problem Statement

We want a mechanism that:

- Tracks task progress across turns
- Validates whether the current intended action is legal
- Delays write actions until prerequisites are satisfied
- Suggests the correct service family when the agent chooses incorrectly
- Does not break existing evaluator / golden-path replay

**Research Question:**  
> How can we augment tool-calling agents with explicit workflow control while preserving benchmark compatibility and keeping the interface simple for weaker models?

---

## 3. Key Design Decisions

### 3.1 Placement of the State Machine

**Option A: System/Orchestrator Layer**
- Global visibility
- Highly invasive to τ²-Bench
- Breaks evaluator compatibility

**Option B: Tool Layer (Chosen)**
- Integrated alongside domain tools
- Can directly guard write tools
- Minimal changes to infrastructure

**Final Decision:** Tool layer

---

### 3.2 Evaluator Compatibility

**Problem:**  
Evaluator replays expected write actions directly on the database.

**Solution: Lazy Activation**

- SM enforcement is **disabled by default**
- Activated only when agent calls `retail_sm_*` tools
- Golden replay never triggers SM → unaffected

---

### 3.3 Agent Engagement Strategy

**Options:**
- Soft Gate → advisory only
- Hard Gate → enforce constraints

**Final Decision:** Hard Gate

- Prevents invalid sequences
- Forces correct workflow
- Especially important for small models

---

### 3.4 Confirmation Strategy

**Options:**
- Per-goal confirmation
- Plan-level confirmation

**Final Decision:** Plan-level confirmation

- Simpler interaction model
- Fewer tool calls
- Matches user mental model

---

### 3.5 SM Output Design

**Problem:** Verbose outputs overwhelm models

**Final Decision: Compact Outputs**

Expose structured fields only:
- `missing_fields`
- `validation_errors`
- `plan_conflicts`

Avoid verbose explanations unless necessary.

---

### 3.6 Read Tool Redesign

**Problem:** Large product outputs overwhelm context

**Solution: Middle-layer tools**

- `get_product_details(product_id)` → metadata only
- `get_variant_details(product_id, filters_json)` → filtered variants

---

## 4. Architecture

### Components

- **Agent**
- **User Simulator**
- **Environment (tools + DB)**
- **Retail State Machine**

### Workflow

1. Agent identifies user
2. Agent activates SM via `retail_sm_set_user`
3. Agent adds goals
4. Agent collects missing information
5. Agent validates plan
6. Agent requests confirmation
7. Agent confirms plan
8. Write tools unlock
9. Agent executes actions

---

## 5. State Machine Model

### Goal-Level State

Each goal contains:

- Goal type
- Order ID (if applicable)
- Filled fields
- Missing fields
- Validation errors
- Plan conflicts

### Plan-Level State

- Authenticated user
- List of goals
- `plan_ok`
- `plan_confirmed`
- Enforcement status
- Mismatch hints

### Phases

- `PLAN`
- `COLLECT`
- `VALIDATE`
- `EXECUTE`

### Enforcement Rules

A write is allowed only if:

- Enforcement is active
- `plan_ok == true`
- `plan_confirmed == true`
- A matching valid goal exists

---

## 6. Research Hypotheses

- Explicit workflow control improves correctness
- Hard gating > soft prompting
- Compact outputs > verbose outputs for small models
- Middle-layer tools improve context efficiency
- SM can improve reliability without modifying evaluator

---

## 7. Risks

- Over-engineering SM interface
- Deadlocks due to strict enforcement
- SM state diverging from DB state
- Breaking evaluator unintentionally
- Tool schema too complex for small models

---

## 8. Success Criteria

### Engineering

- Evaluator unchanged
- Golden replay passes
- Invalid writes are blocked
- Full support for retail actions

### Research

- Higher pass rate vs baseline
- Fewer invalid tool sequences
- More interpretable failures
- Improved robustness on smaller models