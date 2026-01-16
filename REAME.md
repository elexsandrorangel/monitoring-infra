# Monitoring Infrastructure – Ansible Project

This repository contains Ansible playbooks and roles to deploy a basic monitoring stack, including:

- **Alloy** – metrics/logs/telemetry agent
- **Prometheus** – metrics collection and alerting
- **Grafana** – dashboards and visualization

The project is organized to let you provision and update these components on your infrastructure using standard Ansible workflows.

---

## Repository Structure

- `hosts.ini` – Inventory file defining your target hosts and host groups.
- `requirements.yml` – External Ansible roles/collections to install (via `ansible-galaxy install -r requirements.yml`).
- `playbooks/`
  - `alloy.yml` – Playbook to deploy and configure the Alloy agent role.
  - `prometheus.yml` – Playbook to deploy and configure Prometheus.
  - `grafana_servers.yml` – Playbook to deploy and configure Grafana server(s).
  - `notify.yml` – Playbook for notification-related tasks (e.g. handlers, webhooks, alert routing).
  - `update.yml` – Playbook for updating existing monitoring components.
  - `roles/`
    - `alloy/`
    - `prometheus/`
    - `grafana_server/`

---

## How to Use

1. **Install dependencies**

   ```bash
   ansible-galaxy install -r requirements.yml
   ```

2. **Check your inventory**

   Edit `hosts.ini` to define your monitoring hosts, for example:

   ```ini
   [alloy_agents]
   node1
   node2

   [prometheus_servers]
   prom1

   [grafana_servers]
   graf1
   ```

3. **Run a playbook**

   ```bash
   # Deploy Alloy agents
   ansible-playbook -i hosts.ini playbooks/alloy.yml

   # Deploy Prometheus
   ansible-playbook -i hosts.ini playbooks/prometheus.yml

   # Deploy Grafana
   ansible-playbook -i hosts.ini playbooks/grafana_servers.yml

   # Apply updates
   ansible-playbook -i hosts.ini playbooks/update.yml
   ```

Adjust group names, variables, and targeting flags as needed for your environment.

---

## Roles Overview

### Role: `alloy`

**Path:** `playbooks/roles/alloy/`  
**Used by:** `playbooks/alloy.yml`

This role manages installation and configuration of the **Alloy** agent on monitoring targets. Its main responsibilities typically include:

- Installing the Alloy binary/package (from a repository, URL, or local artifact).
- Creating required system user/group and directories (config, data, logs).
- Deploying the Alloy configuration file(s).
- Managing the Alloy service (systemd service creation, start/enable, restart on config change).

**Typical variables (examples):**

These are example variable names you can expect or define in `group_vars`, `host_vars`, or role defaults:

- `alloy_version` – Version of Alloy to install.
- `prometheus_server` –   Points to the base URL of the Prometheus server that other components (e.g. Alloy, Grafana, alerting integrations) should use to send data or query metrics.

**How to use:**

In a playbook:
```yaml
---
- name: Install Alloy on all Alloy hosts
  hosts: alloy
  become: true
  roles:
    - role: alloy
      vars:
        alloy_version: "1.12.0"
        prometheus_server: "https://prometheus.example.com"
```
---

---

### Role: `prometheus`

**Path:** `playbooks/roles/prometheus/`  
**Used by:** `playbooks/prometheus.yml`

This role deploys **Prometheus** as the central metrics and alerting engine.

From the task files layout you can expect responsibilities like:

- **Installation**
  - Downloading/installing the Prometheus binary or package.
  - Creating directories for configuration, data, rules, and logs.
  - Setting up the Prometheus system user and permissions.

- **Configuration**
  - Generating `prometheus.yml` and additional configuration snippets from templates.
  - Managing scrape configurations for:
    - Local exporters
    - Remote Alloy agents
    - Other monitored services
  - Optionally managing alerting rules (e.g., `*.rules.yml` files).

- **Integration with Alloy**
  - A dedicated task file such as `configure-alloy-prometheus.yml` suggests:
    - Updating Prometheus scrape configs to pull metrics exposed by Alloy.
    - Optionally wiring up Alloy to remote_write/remote_read targets, if configured.
    - Ensuring labels and job names align with Alloy’s exported endpoints.

- **Service management**
  - Creating and managing a systemd unit for Prometheus.
  - Starting, enabling, and reloading the service on configuration changes.

**Typical variables (examples):**


- `prometheus_version` – Prometheus version to install.
- `prometheus_user` – System user under which the Prometheus service runs.
- `prometheus_dir` – Directory containing Prometheus configuration files.
- `prometheus_data_dir` – Directory where Prometheus stores its time‑series data.
- 
**How to use:**

```yaml
---
- name: Install Prometheus Server
  hosts: prometheus
  become: true
  roles:
    - role: prometheus
      vars:
        prometheus_version: "2.52.0"
        prometheus_user: prometheus
        prometheus_dir: /etc/prometheus
        prometheus_data_dir: /var/lib/prometheus
```
---

### Role: `grafana_server`

**Path:** `playbooks/roles/grafana_server/`  
**Used by:** `playbooks/grafana_servers.yml`

This role manages **Grafana** server deployment and configuration.

Common responsibilities include:

- Installing Grafana from package repositories or a specified source.
- Configuring:
  - Main `grafana.ini` (server port, domain, auth options, paths).
  - Data sources (e.g., pointing to the Prometheus server).
  - Dashboards (importing JSON dashboards, provisioning folders).
  - Organization and user settings, if desired.
- Managing the Grafana systemd service (enable, start, restart on changes).

**Typical variables (examples):**


- `grafana_version` – Grafana version to install.
- `grafana_user` – System user under which the Grafana service runs.
- `grafana_dir` – Directory containing Grafana configuration files.
- `grafana_data_dir` – Directory where Grafana stores its data (e.g. dashboards, state).

---

## Playbooks

### `playbooks/alloy.yml`

- Targets hosts where you want the Alloy agent installed.
- Typically includes:
  - Role: `alloy`
  - Group: `alloy` (adjust as needed).

## Common Commands

- Run only a specific component with tags (if tags are defined in the roles):

  ```bash
  ansible-playbook -i hosts.ini playbooks/prometheus.yml --tags "config"
  ```

- Limit the run to a specific host:

  ```bash
  ansible-playbook -i hosts.ini playbooks/alloy.yml --limit node1
  ```

---

## Extending the Project

- Add more exporters or services by:
  - Defining new inventory groups.
  - Extending `prometheus_scrape_configs` variables.
  - Optionally adding new roles for exporters.
- Enhance dashboards by:
  - Adding dashboard JSON files.
  - Extending `grafana_dashboards` variables or provisioning configuration.
- Add alerting:
  - Define alert rules in the Prometheus role.
  - Configure alertmanager via a new role/playbook and integrate it into this stack.

---

## Requirements

- Ansible (version compatible with your roles, e.g. 2.12+).
- SSH access to all managed hosts.
- Sudo privileges for tasks that require elevated permissions (e.g., package installation, service management).

---