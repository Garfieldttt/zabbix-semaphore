# Zabbix Template Semaphore Ansible Task Monitoring

Agentless monitoring of Ansible Semaphore via REST API. Dynamically discovers templates via LLD, monitors task status, start/end time, playbook name, and app type. Triggers alerts on failures and stuck running tasks.

---

## Requirements

- **Zabbix Server** version 7.0 or higher
- Ansible Semaphore instance with API access
- Valid API token set in macro `{$API_TOKEN}`
- Host macros configured on the Zabbix host object (see Macros section)

---

## Create an API Token

1. Open the Semaphore web interface and log in.
2. Navigate to **User → API Tokens** → **Create Token**.
3. Copy the token - it is only shown once.

---

## Installation

1. **Import into Zabbix**
   - Go to **Configuration → Templates → Import** and select the YAML file from the `7.0/` directory.

2. **Create a Host**
   - Go to **Configuration → Hosts → Create host**
     - **Host name:** `semaphore01`
     - **Groups:** e.g. `Applications` - do **not** use `Templates` (reserved for Zabbix template objects)
     - **Interfaces:** leave empty
     - **Templates:** attach `Zabbix template semaphore task monitoring`

3. **Configure Macros**
   - Open the **Macros** tab on the host and set the required macros (see below).

---

## Macros

### Required

| Macro               | Example value                | Description                                              |
|---------------------|------------------------------|----------------------------------------------------------|
| `{$API_TOKEN}`      | *(SECRET_TEXT)*              | Semaphore API token (Bearer)                             |
| `{$SEMAPHORE_URL}`  | `http://192.168.10.105:3000` | Base URL of the API including port - **no trailing slash** |
| `{$PROJECT_NUMBER}` | `1`                          | Project ID - visible in the Semaphore URL (`/project/1/`) |

### Optional

| Macro                       | Default  | Description                                                                                     |
|-----------------------------|----------|-------------------------------------------------------------------------------------------------|
| `{$ENABLE_TRIGGER}`         | `1`      | Enable task failure alerts globally (1 = on, 0 = off)                                          |
| `{$ENABLE_TRIGGER:"{#ID}"}` | *(inherits)* | Per-template override via context macro - set `{$ENABLE_TRIGGER:"3"}=0` on the host to silence a specific template |
| `{$JOB_RUNNING_TIME}`       | `1h`     | Time period with unit (e.g. `1h`, `30m`) - alert fires when task is continuously in "running" state for this duration |

---

## Template Contents

### HTTP Agent Items

| Key | Endpoint | Interval | Description |
|-----|----------|----------|-------------|
| `semaphore.ping` | `GET /api/ping` | 60s | Health check - expects `"pong"` |
| `semaphore.info.raw` | `GET /api/info` | 3600s | Raw version info |
| `semaphore.raw` | `GET /api/project/{$PROJECT_NUMBER}/templates` | 300s | Template list - source for LLD and all dependent items |

### Dependent Items (from `semaphore.info.raw`)

| Key | Description |
|-----|-------------|
| `semaphore.version` | Semaphore software version |
| `semaphore.ansible.version` | Ansible version on the Semaphore host |

### Discovery Rule

**semaphor-discover** - dependent on `semaphore.raw`

LLD macros per discovered template:

| Macro             | Field            | Description                              |
|-------------------|------------------|------------------------------------------|
| `{#ID}`           | `$.id`           | Template ID                              |
| `{#NAME}`         | `$.name`         | Template name                            |
| `{#LAST_TASK_ID}` | `$.last_task.id` | Last task run ID                         |
| `{#STATUS}`       | `$.status`       | Top-level template status (informational) |

### Item Prototypes (all dependent on `semaphore.raw`)

| Key | Description |
|-----|-------------|
| `task.status[{#ID}]` | Last task status as a number (valuemap: 0–4) |
| `task.starttime[{#ID}]` | Start time of the last run (Unix timestamp) |
| `task.endtime[{#ID}]` | End time of the last run - discarded while task is running |
| `task.playbook[{#ID}]` | Playbook name of the last run |
| `task.app[{#ID}]` | App type of the template (ansible, terraform, …) |

### Triggers

| Name | Condition | Severity |
|------|-----------|----------|
| Semaphore API unreachable | `semaphore.ping <> "pong"` | Average |
| Task `{#NAME}` failed | `task.status = 2` and `{$ENABLE_TRIGGER:"{#ID}"}=1` | Warning |
| Task `{#NAME}` running longer than `{$JOB_RUNNING_TIME}` | `task.status = 3` continuously for `{$JOB_RUNNING_TIME}` | Warning |

All triggers: Manual close = Yes.

---

## Dashboard

The template includes a **Semaphore Task Monitoring** dashboard with:
- **Problems widget** (full width) - all active alerts for this host
- **Item widget** - API status (ping)
- **Item widget** - Semaphore version
- **Item widget** - Ansible version
- **Clock widget** - server time

---

## Value Map - Task-status

| Value | Status  |
|-------|---------|
| `0`   | stopped |
| `1`   | success |
| `2`   | error   |
| `3`   | running |
| `4`   | waiting |

---

## Multiple Projects

Create one host per Semaphore project and set `{$PROJECT_NUMBER}` individually per host. No template changes required.

---

> **Source:** https://github.com/Garfieldttt/zabbix-semaphore-ansible/tree/zabbix
