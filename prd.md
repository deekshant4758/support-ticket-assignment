# Product Requirements Document: Support Ticket Assignment Service

## 1. Overview
This document outlines the product requirements for an automated Support Ticket Assignment Service. The service replaces manual ticket triaging by team leads, ensuring tickets are instantly assigned to available agents, workload is distributed fairly, and coverage gaps are visible to management.

## 2. Problem Statement
As support teams scale, manual ticket assignment by team leads breaks down. Leads become bottlenecks, tickets sit unassigned when leads are offline, and workload distribution becomes highly uneven, leading to agent burnout and poor customer response times.

## 3. Target Users
* **Team Leads / Admins:** Configure team availability, monitor coverage gaps, and understand assignment logic.
* **Support Agents:** Receive a fair, balanced workload without being overwhelmed when already at capacity.
* **Customers (Indirect):** Expect immediate ownership of their tickets and timely responses.

## 4. Scope & Non-Goals
**In Scope:**
* UI for managing agent profiles and recurring weekly schedules (including timezones).
* UI for visualizing team coverage and identifying scheduling gaps.
* Automated assignment logic based on real-time availability and fairness.
* Transparency features to explain assignment decisions.

**Out of Scope (Explicitly excluded):**
* User authentication, roles, or permissions (assume the UI user is authorized).
* Billing, account management, or company onboarding.
* Holiday calendars, one-off schedule overrides, or time-off requests.
* Third-party integrations (PagerDuty, Slack, etc.).
* Mobile interfaces.

## 5. Core Definitions & Logic
To ensure clarity, the following definitions govern the system's behavior:

* **Availability:** An agent is "available" if the current time falls within their recurring weekly schedule in their local IANA timezone.
  * **Overnight Shifts:** Supported (e.g., 22:00–06:00). Weekdays are determined strictly by local wall-clock time.
  * **DST Handling:** Shifts are defined in local wall-clock time. DST gaps/folds are resolved by converting local boundaries to UTC via standard IANA rules, automatically adjusting the shift's absolute duration.
* **Fairness (Load Balancing):** Work is distributed by assigning tickets to the available agent with the lowest number of active tickets. 
* **Tie-Breaker (Recency):** If multiple available agents have the same lowest active ticket count, the ticket goes to the agent who was assigned a ticket least recently.
* **Active Ticket:** A ticket currently assigned to an agent that has not been marked as resolved or closed.
* **Target Capacity (Soft Limit):** A preferred maximum number of active tickets for an agent. The system will prioritize assigning tickets to agents under this limit. If all available agents are over this limit, the system will still assign the ticket to the agent with the lowest load to ensure no ticket is left unowned.
* **Coverage Gap:** A time window where zero agents on the team are scheduled to be available.

## 6. User Stories

### Team Lead / Admin
* **Schedule Setup:** As a Team Lead, I want to define my agents' recurring weekly schedules and timezones so that the system knows exactly when they are working.
* **Capacity Limits:** As a Team Lead, I want to set a maximum active ticket capacity for each agent so they don't get overwhelmed with too much concurrent work.
* **Coverage Visibility:** As a Team Lead, I want to view a visual timeline of team coverage so I can easily spot gaps where no one is scheduled to work and hire/adjust schedules accordingly.
* **Assignment Transparency:** As a Team Lead, I want to see a clear explanation of why a specific ticket was assigned to a specific agent so I can understand and trust the automated logic.

### Support Agent
* **Fair Workload:** As a Support Agent, I want tickets to be distributed evenly among available peers so that the workload is fair across the team.
* **Capacity Protection:** As a Support Agent, I want the system to prioritize giving new tickets to agents who are under their target capacity, and alert my Team Lead when we are all overloaded, so that we can manage expectations and adjust schedules before burnout occurs.


### Customer (Indirect)
* **Immediate Ownership:** As a Customer, I want my support ticket to be immediately assigned to a team member so that I know my issue is being actively handled.

## 7. Functional Requirements

### 7.1. Availability & Coverage Management (UI)
The UI allows Team Leads to configure the team and visualize their coverage.
* **Agent Management:** View a list of agents, add/remove agents, and set their local timezones.
* **Schedule Management:** Define a recurring weekly schedule for each agent (e.g., Monday 09:00 - 17:00) and define their "Max Active Tickets" capacity.
* **Coverage Visualization:** A timeline view showing overall team coverage. It must visually highlight "Coverage Gaps" (no agents scheduled) and "Capacity Warnings" (scheduled agents likely to exceed max capacity based on current load).

### 7.2. Automated Ticket Assignment
* **Assignment Contract:** The assignment operation accepts `company_id` and `ticket_id`.
  * **Success Outcome:** Returns the assigned agent, the assignment reason, and confirms the ticket is now owned.
  * **No-Assignment Outcome:** If the company has zero agents configured, the operation fails and returns an "Unassignable" error, preventing the ticket from entering a void.
  * **Idempotency (Retries):** If the system receives a request for a `ticket_id` that has already been assigned, it must return the *original* assignment details. It must not create a duplicate assignment, nor should it alter the agent's active ticket count or advance the fairness/recency state.
* **Assignment Logic:** Evaluate all agents for the company. Filter for those who are currently "Available". Sort the available agents by: 
  1. Agents currently *under* their Target Capacity (preferred).
  2. Lowest active ticket count.
  3. Least recently assigned.
  Assign to the top agent. *(Note: If all available agents are over their Target Capacity, the system ignores the capacity preference and simply assigns to the available agent with the absolute lowest load).*

### 7.3. Ticket Resolution Tracking
* **Resolution Flow:** The system provides a dedicated resolution API endpoint (e.g., `POST /tickets/{ticket_id}/resolve`), alongside a "Resolve" action in the UI, to mark tickets as closed.
* **Duplicate-Close Behavior:** The resolution action is strictly idempotent. If a ticket is marked as resolved but is already in a "resolved" state, the system treats it as a no-op. It will not decrement the active ticket count below zero, nor will it trigger duplicate state changes or fairness adjustments.

## 8. Edge Cases & Product Behavior

| Scenario | Product Behavior |
| :--- | :--- |
| **No agents are currently available** | The system will still assign the ticket to prevent it from sitting unassigned. It will choose the agent with the lowest overall workload and flag the assignment as "out of hours" for the Team Lead to review. |
| **All available agents are over Target Capacity** | The system prioritizes ticket ownership over capacity limits. It will assign the ticket to the available agent with the lowest overall active load. The assignment is explicitly flagged as "Over Capacity" in the UI so the Team Lead knows the team is overloaded and needs to adjust schedules or hire. |
| **Empty team** | The system will reject the assignment and alert the Team Lead that the team has no agents configured, preventing tickets from falling into a void. |
| **Agent's timezone changes** | The system will immediately recalculate availability based on the new timezone for all future assignments. |

## 9. Assumptions
1. **Data Seeding:** Companies, agents, and initial tickets are pre-seeded or managed via a separate external system. We only manage the *availability* and *assignment* logic.
2. **State Management:** The service will maintain an internal state for `active_ticket_counts` per agent to calculate fairness. 
3. **Time Accuracy:** The system relies on accurate server time to correctly evaluate timezone-based availability.
