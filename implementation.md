# Technical Design: Support Ticket Assignment Service

## 1. Data Model

```mermaid
erDiagram
    COMPANY ||--o{ AGENT : has
    COMPANY ||--o{ TICKET : has
    AGENT ||--o{ AVAILABILITY_BLOCK : has
    AGENT ||--o{ ASSIGNMENT : receives
    TICKET ||--o| ASSIGNMENT : "has (0 or 1)"

    COMPANY { uuid id, string name, string display_timezone }
    AGENT { uuid id, uuid company_id, string name, string timezone, int target_capacity, string status, datetime last_assigned_at }
    AVAILABILITY_BLOCK { uuid id, uuid agent_id, int day_of_week, int start_minute, int end_minute }
    TICKET { uuid id, uuid company_id, datetime created_at, string status }
    ASSIGNMENT { uuid id, uuid ticket_id, uuid agent_id, datetime assigned_at, string reason, bool out_of_hours, bool over_capacity }
```

**Key Design Choices:**
- **UUIDs**: Generated at the application layer to ensure safe merging if the database is ever migrated.
- **Status Constraints**: Enforced via SQLite `CHECK` constraints (`active`/`inactive` for agents, `new`/`open`/`resolved` for tickets) to prevent invalid state drift.
- **Derived Active Count**: We do not store a mutable `active_ticket_count` on the agent. It is derived via `SELECT COUNT(*) FROM assignment JOIN ticket WHERE agent_id = ? AND ticket.status = 'open'`. This guarantees the count can never drift out of sync with reality.
- **Separate ASSIGNMENT Table**: Holds assignment-specific metadata (`reason`, `out_of_hours`, `over_capacity`) without polluting the `TICKET` table. Enforced 1:1 via `UNIQUE` constraint on `assignment.ticket_id`.
- **Overlapping Blocks**: The UI allows drag-to-select, but the `PUT /availability` endpoint merges any overlapping or adjacent intervals into canonical, non-overlapping blocks before saving to the database.

## 2. Core Algorithms

### 2.1 Availability Check
We evaluate availability by converting the **current UTC instant** to the **agent's local wall-clock time** (using Luxon). 
- We check if this local time falls within any of the agent's merged availability blocks.
- **Overnight shifts** (e.g., `start_minute > end_minute`) are handled by checking if the local time is either on the start day after the start time, or on the following day before the end time.
- **DST** is handled natively: because we evaluate a specific UTC instant into a local time, we never construct invalid local times (spring-forward gaps) or ambiguous ones (fall-back folds).

### 2.2 Agent Selection & Fairness
1. **Filter**: Get all `active` agents for the company.
2. **Tiering**: 
   - *Tier 1 (Normal)*: Agents currently available AND under `target_capacity`.
   - *Tier 2 (Over Capacity)*: Agents currently available BUT at or above `target_capacity`.
   - *Tier 3 (Out of Hours)*: All active agents (ignoring schedule entirely).
3. **Sorting**: Within the highest available tier, sort by **utilization rate** (`activeCount / target_capacity`) ascending, then by `last_assigned_at` ascending (nulls first). 
   - *Why utilization rate?* Sorting by raw count unfairly penalizes agents with higher capacities. Utilization ensures load is distributed proportionally to each agent's defined capacity.
4. **Fallback**: If Tier 1 is empty, use Tier 2. If Tier 2 is empty, use Tier 3. If no active agents exist, throw `Unassignable`.

### 2.3 Idempotency
- **Assignment**: The endpoint first validates that the `ticket_id` belongs to the provided `company_id`. If it does, it checks for an existing assignment. If found, it returns the existing assignment details immediately without mutating any state (no new rows, no `last_assigned_at` update).
- **Resolution**: If `ticket.status === 'resolved'`, the endpoint returns a 200 OK no-op. The derived active count automatically reflects the change without manual decrementing.

## 3. API Design

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/companies/:id/agents` | List agents and their current derived active load. |
| `POST`| `/api/companies/:id/agents` | Create agent (`name`, `timezone`, `target_capacity`). |
| `PUT` | `/api/agents/:id/availability`| Replace weekly schedule. Accepts array of blocks. Overlaps are merged server-side. |
| `GET` | `/api/companies/:id/coverage` | Returns discrete time slots for the weekly heatmap (see §3.1). |
| `POST`| `/api/tickets/:id/assignment`| Body: `{ company_id }`. Returns assigned agent, reason, and flags. Idempotent. |
| `POST`| `/api/tickets/:id/resolve` | Marks ticket resolved. Idempotent. |

### 3.1 Coverage Endpoint & Slots
**Why slots?** The UI requires a discrete, performant heatmap to visually highlight gaps and capacity warnings across a 7-day week. Calculating this on the fly per pixel is inefficient; pre-computed discrete slots provide a clean contract between backend and frontend.

**Partial Coverage Handling**: A time slot is *only* marked as covered by an agent if the agent's availability block **fully encompasses** the slot's entire duration. For example, if granularity is 30 minutes, a 9:00–9:45 block will mark the 9:00–9:30 slot as covered, but will *not* mark the 9:30–10:00 slot, preventing false coverage signals.

**Response Shape**:
```json
{
  "slots": [
    { 
      "day_of_week": 1, 
      "start_minute": 540, 
      "available_agent_ids": ["uuid-1", "uuid-2"], 
      "gap": false, 
      "capacity_warning": true 
    }
  ]
}
```
- `gap`: `true` if `available_agent_ids` is empty.
- `capacity_warning`: `true` if `available_agent_ids` is not empty, BUT every agent in that list currently has `activeCount >= target_capacity`. (Mutually exclusive with `gap`).

## 4. Main UI Flow

1. **Team & Schedule**: Manage agents (name, timezone, target capacity). A 7-day grid allows click-and-drag scheduling. Overnight blocks are supported by dragging across midnight. 
2. **Coverage Heatmap**: A read-only weekly grid colored by slot status: Normal (green/dark), Capacity Warning (amber/hatched), or Gap (red). Hovering a cell shows which agents are scheduled and their current load vs. target.
3. **Tickets**: List of tickets with status, assigned agent, and an expandable row showing the exact `reason` string and `out_of_hours`/`over_capacity` badges. Includes a dev-only "Simulate New Ticket" button to demonstrate the assignment flow end-to-end.

## 5. Edge Cases

| Scenario | Implementation Behavior |
| :--- | :--- |
| **Idempotent assignment with mismatched `company_id`** | The API validates ticket ownership *before* checking for idempotent replay. A request for Company B's ID on a ticket belonging to Company A returns `404`, preventing data leakage. |
| **DST Transitions** | Because availability checks convert a known UTC instant to local time (rather than constructing local times and converting to UTC), spring-forward gaps and fall-back folds are resolved natively by the timezone library without special-casing. |
| **All agents are `inactive`** | The selection algorithm filters for `status === 'active'` first. An all-inactive team is treated identically to an empty team, returning a `409 Unassignable` error. |
| **Resolution called multiple times** | The endpoint checks `status === 'resolved'` and returns a no-op. Because the active count is a derived `COUNT()` query, it naturally reflects the correct state without risk of decrementing below zero. |
| **Agent timezone changed mid-day** | Availability is calculated dynamically at the exact moment of the assignment API call using the current database value. The new timezone takes effect immediately for the next ticket. |

## 6. Test Plan

- **Unit**: 
  - `isAvailableAt`: Standard blocks, overnight blocks (specifically testing Sunday 22:00 to Monday 06:00 to verify day-of-week wrapping), and DST boundary instants.
  - `selectAgent`: Verifies tier fallback (Normal → Over Capacity → Out of Hours) and utilization-rate sorting with `last_assigned_at` tie-breaking.
- **Integration**: 
  - Assignment idempotency: Two identical calls yield one DB row and identical responses.
  - Security: Assignment call with valid ticket but wrong `company_id` returns `404`.
  - Resolution idempotency: Two calls yield one status change; derived count updates correctly.
  - Coverage: Verifies `gap` and `capacity_warning` are mutually exclusive and correctly calculated.
- **Manual/UI**: Dragging an overnight block saves correctly; heatmap colors match API slot data; simulated ticket assignment displays the correct reason string in the UI.