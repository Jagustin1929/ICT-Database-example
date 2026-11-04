# ICTSAS432 Help Desk Database and Service Desk Console

A synthetic help desk database for teaching and assessing **ICTSAS432 - Identify and resolve client ICT problems**. It lets students work through realistic tickets: log them, assign them to technicians, troubleshoot, escalate between support tiers, and resolve them.

All people, devices, emails, phone numbers and events are fictional. The company is "Harbourview Group" with offices in Sydney CBD, Parramatta and Wollongong.

## What is in this repository

| File | What it is | Needs |
|---|---|---|
| `service_desk_console.html` | A point-and-click help desk app with the database built in | A web browser (Edge or Chrome). No install |
| `ictsas432_helpdesk.db` | The SQLite database | A SQLite tool, such as DB Browser for SQLite or VS Code with a SQLite extension |
| `ictsas432_helpdesk.sql` | SQL script that rebuilds the SQLite database | Any SQLite tool |
| `ictsas432_helpdesk_mysql.sql` | MySQL 8.0 version of the database | MySQL Workbench or phpMyAdmin |

## Quick start

### Option 1: the service desk console (easiest)
1. Download `service_desk_console.html`.
2. Double-click it to open it in Edge or Chrome.
3. Pick who you are in **Working as**, then click a ticket to work on it.

No installation, admin rights or internet connection is needed. Your changes are kept in the browser automatically. Click **Save database** to write a `.db` file you can reopen later with **More > Open a .db file**. **More > Reset** returns to the original data.

### Option 2: work with the SQLite database directly
Open `ictsas432_helpdesk.db` in any of these:
- **DB Browser for SQLite** (the portable .zip version needs no admin rights)
- **VS Code** with a SQLite viewer extension
- an online SQLite viewer (the data is synthetic, so uploading it is safe)

### Option 3: MySQL
1. Install MySQL Community and MySQL Workbench (or use WampServer with phpMyAdmin).
2. In Workbench, choose **File > Open SQL Script**, select `ictsas432_helpdesk_mysql.sql` and click the lightning bolt.
3. In phpMyAdmin, use the **Import** tab instead.
4. The script creates the database `ictsas432_helpdesk`. Running it again resets everything.

The MySQL script is tested on MySQL 8.0. It has not been tested on MariaDB.

## What the data contains

| Table | Rows | Purpose |
|---|---|---|
| `tickets` | 134 | Incidents and service requests (110 resolved or closed, 24 open) |
| `clients`, `departments` | 60, 8 | People who report problems |
| `staff` | 12 | 4 Tier 1, 4 Tier 2, 3 Tier 3 and a Service Desk Manager |
| `assets` | 176 | Laptops, desktops, monitors, docks, printers, phones |
| `ticket_assignments` | 244 | Full history of who owned each ticket |
| `escalations` | 97 | Every escalation, with reason and hand-over information |
| `ticket_notes` | 1,232 | Troubleshooting, client contact and resolution notes |
| `knowledge_base`, `ticket_kb` | 26, 73 | Fix articles and which tickets used them |
| `categories`, `priorities`, `statuses`, `config` | | Lookup tables and settings |

**Views:** `v_open_tickets`, `v_escalation_history`, `v_staff_workload`, `v_sla_performance`.

**Support tiers:** Tier 1 (service desk analysts), Tier 2 (desktop, systems and network technicians), Tier 3 (specialist engineers and security), and a manager for hierarchical escalations.

### Priorities and deadlines
Deadlines are counted in business hours only (Monday to Friday, 8am to 5pm).

| Priority | Respond within | Resolve within |
|---|---|---|
| P1 Critical | 15 minutes | 4 hours |
| P2 High | 30 minutes | 8 hours |
| P3 Medium | 1 hour | 24 hours |
| P4 Low | 4 hours | 40 hours |

A ticket is **Overdue** when its resolve-by time has passed and it is still open. The database uses a simulated clock stored in `config` (`reference_datetime`, 3 Nov 2026 09:30) so the data always looks the same.

## The service desk console

- **Tickets:** filter and search the queue. The summary cards show open tickets, tickets with no owner, escalated tickets waiting to be picked up, and overdue tickets.
- **Ticket detail:** client, device, deadlines, notes, assignment history, escalation trail and knowledge base articles. A strip shows everyone who has held the ticket.
- **Actions:** Log ticket, Assign or Reassign, Escalate, Add note, Change status, Resolve and Close.
- **Technicians:** workload for each person and warnings for staff on leave.
- **Reports:** 11 ready-made reports, including open cyber tickets that were escalated and open tickets owned by staff on leave. Download them as CSV or print them.
- **SQL:** run your own queries, and see the SQL behind every button you press.

Assigning suggests the technician with the fewest open tickets who is not on leave. This looks at workload and availability only, so students still need to judge who has the right skills.

## Escalating and assigning with SQL

### SQLite
Triggers keep the ticket's owner, tier and status up to date.

```sql
-- Assign a ticket, or reassign it (closes the previous assignment)
INSERT INTO ticket_assignments (ticket_id, assigned_to, assigned_by, assigned_at, reason)
VALUES ((SELECT ticket_id FROM tickets WHERE ticket_number = 'INC-100131'), 2, 12, '2026-11-03 09:40:00', 'Assigned by manager');

-- Escalate from the current owner to another person
INSERT INTO escalations (ticket_id, escalation_type, from_staff_id, to_staff_id, from_tier, to_tier, escalated_at, reason, info_provided)
VALUES ((SELECT ticket_id FROM tickets WHERE ticket_number = 'INC-100131'), 'Functional', 2, 5, 1, 2,
        '2026-11-03 10:00:00', 'Needs admin rights', 'Steps tried: restarted spooler, cleared queue');
```

Escalation types are `Functional` (to a higher tier or specialist), `Hierarchical` (to the manager) and `Reassignment` (manager hands it down). Assigning to inactive staff is blocked.

### MySQL
MySQL does not let a trigger change its own table, so use the stored procedures instead:

```sql
CALL sp_assign_ticket('INC-100131', 2, 12, 'Assigned by manager');
CALL sp_escalate_ticket('INC-100131', 5, 'Functional', 'Needs admin rights', 'Steps tried: restarted spooler');
```

The MySQL version stores `config` as `config_key` and `config_value`, because `key` is a reserved word.

## Handy queries

```sql
SELECT * FROM v_open_tickets ORDER BY priority, created_at;              -- open queue with deadline status
SELECT * FROM v_open_tickets WHERE assigned_to = 'UNASSIGNED';           -- tickets with no owner
SELECT * FROM v_staff_workload ORDER BY open_tickets DESC;               -- workload and who is on leave
SELECT * FROM v_escalation_history WHERE ticket_number = 'INC-100103';   -- escalation trail
SELECT * FROM v_sla_performance;                                         -- percent resolved within deadline
```

Open cyber tickets that have been escalated:

```sql
SELECT t.ticket_number, t.summary, p.code AS priority, s.name AS status,
       COUNT(e.escalation_id) AS times_escalated, MAX(e.escalated_at) AS last_escalated
FROM tickets t
JOIN categories c ON c.category_id = t.category_id
JOIN statuses s ON s.status_id = t.status_id
JOIN priorities p ON p.priority_id = t.priority_id
JOIN escalations e ON e.ticket_id = t.ticket_id
WHERE c.name = 'Security' AND s.is_open = 1
GROUP BY t.ticket_id
ORDER BY p.code, last_escalated DESC;
```

## Built-in scenarios for assessment tasks

- 3 new tickets with no owner, ready to be assigned
- 4 open tickets owned by Jacob Williams, who is on leave until 10 Nov, to be reassigned
- 2 P1 phishing tickets escalated to the manager and past their deadline
- Tickets that are waiting on the client, and tickets waiting to be picked up after escalation
- Resolved tickets that missed their deadline, and tickets passed between people within a tier
- A mix of incidents and service requests across hardware, software, network, accounts, email, printing, security and mobile devices

## Reference ids

- **Statuses:** 1 New, 2 Assigned, 3 In Progress, 4 Awaiting Client, 5 Escalated, 6 Resolved, 7 Closed
- **Priorities:** 1 = P1, 2 = P2, 3 = P3, 4 = P4
- **Staff:** 1-4 Tier 1, 5-8 Tier 2, 9-11 Tier 3, 12 Service Desk Manager

## Notes and limits

- The console runs the database in the browser using [sql.js](https://github.com/sql-js/sql.js) (SQLite compiled to WebAssembly, MIT licence), which is bundled inside the HTML file.
- Work saved "in the browser" belongs to that browser on that computer. On shared computers, save a `.db` file and use **More > Reset** between users.
- A managed computer could block locally opened HTML files or downloaded tools. Try the console first on the machines you plan to use.
- This is teaching material. It is not a real ticketing system and holds no real data.

## Licence

Add a licence file that suits your use before sharing publicly. The data is entirely synthetic, so it can be reused and adapted freely.
