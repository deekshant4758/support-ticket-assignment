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

* **Availability:** An agent is "available" if the current time (converted to the agent's local timezone) falls within their defined recurring weekly schedule.
* **Fairness (Load Balancing):** Work is distributed by assigning tickets to the available agent with the lowest number of active tickets. 
* **Tie-Breaker (Recency):** If multiple available agents have the same lowest active ticket count, the ticket goes to the agent who was assigned a ticket least recently.
* **Active Ticket:** A ticket currently assigned to an agent that has not been marked as resolved or closed.
* **Coverage Gap:** A time window where zero agents on the team are scheduled to be available.

## 6. User Stories

### Team Lead / Admin
* **Schedule Setup:** As a Team Lead, I want to define my agents' recurring weekly schedules and timezones so that the system knows exactly when they are working.
* **Capacity Limits:** As a Team Lead, I want to set a maximum active ticket capacity for each agent so they don't get overwhelmed with too much concurrent work.
* **Coverage Visibility:** As a Team Lead, I want to view a visual timeline of team coverage so I can easily spot gaps where no one is scheduled to work and hire/adjust schedules accordingly.
* **Assignment Transparency:** As a Team Lead, I want to see a clear explanation of why a specific ticket was assigned to a specific agent so I can understand and trust the automated logic.

### Support Agent
* **Fair Workload:** As a Support Agent, I want tickets to be distributed evenly among available peers so that the workload is fair across the team.
* **Capacity Protection:** As a Support Agent, I want the system to stop assigning me new tickets when I reach my maximum capacity so I can focus on resolving my current queue without falling behind.

### Customer (Indirect)
* **Immediate Ownership:** As a Customer, I want my support ticket to be immediately assigned to a team member so that I know my issue is being actively handled.

## 7. Functional Requirements

### 7.1. Availability & Coverage Management (UI)
The UI allows Team Leads to configure the team and visualize their coverage.
* **Agent Management:** View a list of agents, add/remove agents, and set their local timezones.
* **Schedule Management:** Define a recurring weekly schedule for each agent (e.g., Monday 09:00 - 17:00) and define their "Max Active Tickets" capacity.
* **Coverage Visualization:** A timeline view showing overall team coverage. It must visually highlight "Coverage Gaps" (no agents scheduled) and "Capacity Warnings" (scheduled agents likely to exceed max capacity based on current load).

### 7.2. Automated Ticket Assignment
The system processes incoming tickets and automatically assigns them.
* **Assignment Logic:** Evaluate all agents for the company. Filter for those who are currently "Available" AND under their "Max Active Tickets" limit. Sort by lowest active ticket count, then by least recently assigned. Assign to the top agent.
* **Transparency:** The system must generate and store a human-readable reason for every assignment (e.g., *"Available, lowest active load, least recently assigned"*).

### 7.3. Ticket Resolution Tracking
To maintain accurate "active ticket" counts, the system must track when tickets are closed.
* **Resolution Handling:** The system must listen for or provide a mechanism to mark tickets as resolved, automatically decrementing the assigned agent's active ticket count.

## 8. Edge Cases & Product Behavior

| Scenario | Product Behavior |
| :--- | :--- |
| **No agents are currently available** | The system will still assign the ticket to prevent it from sitting unassigned. It will choose the agent with the lowest overall workload and flag the assignment as "out of hours" for the Team Lead to review. |
| **All available agents are at max capacity** | The system will prioritize availability over capacity limits to ensure the ticket is handled, assigning it to the available agent with the lowest current load. |
| **Empty team** | The system will reject the assignment and alert the Team Lead that the team has no agents configured, preventing tickets from falling into a void. |
| **Agent's timezone changes** | The system will immediately recalculate availability based on the new timezone for all future assignments. |

## 9. Assumptions
1. **Data Seeding:** Companies, agents, and initial tickets are pre-seeded or managed via a separate external system. We only manage the *availability* and *assignment* logic.
2. **State Management:** The service will maintain an internal state for `active_ticket_counts` per agent to calculate fairness. 
3. **Time Accuracy:** The system relies on accurate server time to correctly evaluate timezone-based availability.
