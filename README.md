# Zabbix Integration: Semaphore Ansible Task Monitoring

**Template:** `Zabbix template semaphore task monitoring`  
**Zabbix Version:** 7.0+

---

## 💡 Overview

Agentless monitoring of [Ansible Semaphore](https://ansible-semaphore.com/) via its REST API:

- Fetches all templates from a specific Semaphore project (`{$SEMAPHORE_URL}/api/project/{$PROJECT_NUMBER}/templates`)
- Uses Low-Level Discovery (LLD) to dynamically detect templates
- Retrieves and monitors the status of the last executed task for each template
- Applies JSON preprocessing and value mapping to normalize statuses
- Triggers alerts when tasks fail (configurable per template via macro)

---

## 🔍 Requirements

- Semaphore instance with REST API access
- A valid API token stored in the macro `{$API_TOKEN}`
- API must be reachable via `{$SEMAPHORE_URL}`
- Semaphore project ID `{$PROJECT_NUMBER}`
- Template must be applied to a Zabbix host that can reach the Semaphore API

| Macro               | Example Value                   | Description                                                  |
|---------------------|----------------------------------|--------------------------------------------------------------|
| `{$API_TOKEN}`       | _(SECRET)_                      | Semaphore API token for authentication (Bearer token)        |
| `{$SEMAPHORE_URL}`   | `http://192.168.10.105:3000`    | Base URL of the Semaphore API (including port)               |
| `{$PROJECT_NUMBER}`  | `1`                             | Semaphore project ID (e.g., 1 = first project)               |
| `{$ENABLE_TRIGGER}`  | `1`                             | Enables per-task error triggers (1 = enabled, 0 = disabled)  |

---

## 📑 Items Included in the Template

| Item Key                 | Description                                                              |
|--------------------------|---------------------------------------------------------------------------|
| `semaphore.raw`          | HTTP Agent item fetching template data from the API                      |
| `task.status[{#ID}]`     | Dependent item showing last task status for a discovered template         |

---

## 🔄 Discovery Rule: `semaphor-discover`

Discovers all templates within the specified Semaphore project using LLD.  
Automatically creates:

- Items: `task.status[{#ID}]` for each template
- Triggers: Prototype trigger for task failures

### 🎯 LLD Macros Used

| LLD Macro         | JSON Path             | Purpose                                |
|-------------------|------------------------|----------------------------------------|
| `{#ID}`           | `$.id`                | Template ID                            |
| `{#NAME}`         | `$.name`              | Template name                          |
| `{#STATUS}`       | `$.status`            | Template status (optional)             |
| `{#LAST_TASK_ID}` | `$.last_task.id`      | ID of the last task for the template   |

---

## 🔁 Value Mapping: `Task-status`

Maps string-based status values from the API into Zabbix-readable numeric values:

| Value | Mapped To |
|--------|-----------|
| `0`    | stopped   |
| `1`    | success   |
| `2`    | error     |
| `3`    | running   |

---

## ⚠️ Trigger Prototype

| Name                                          | Expression                                                                 |
|-----------------------------------------------|----------------------------------------------------------------------------|
| `Task {#NAME} Task-ID {#LAST_TASK_ID} failed` | `last(/Template Semaphore/task.status[{#ID}])=2 and {$ENABLE_TRIGGER:"{#ID}"}=1` |

- Trigger fires when the last task for a template ends in error (`error`)
- Triggers can be selectively disabled per task via the macro `{$ENABLE_TRIGGER}`

---

## 📦 Example Macro Configuration

```text
{$API_TOKEN}        = secret_generated_token
{$SEMAPHORE_URL}    = http://192.168.10.105:3000
{$PROJECT_NUMBER}   = 1
{$ENABLE_TRIGGER}   = 1
