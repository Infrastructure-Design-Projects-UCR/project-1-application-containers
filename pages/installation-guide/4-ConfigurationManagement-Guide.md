# Installation guide

## Summary

The last past part of the project was about setting up a **domain controller** with **FreeIPA** solution in our infrastructure. Now we will focus on configuring the **Ansible** tool to manage the infrastructure and automate the deployment of services across the nodes.

This configuration guide gets the domain DNS services already configured to configure the tool of configuration management and deployment automation with **Ansible**.

## Objectives

1. Configure the **Ansible** tool to manage the infrastructure.
   1. Install the **Ansible** package on the control node.
   2. Configure the **Ansible** inventory file to include all managed nodes.
   3. Set up SSH key-based authentication for passwordless access to managed nodes.

2. Create the **Ansible** playbooks to define the desired state of the infrastructure.
   1. Create playbooks to check the network connectivity between required nodes.
   2. Create playbooks to check firewall rules and ensure they are correctly configured.
   3. Create playbooks to check packages needed.
   4. Create playbooks to check dependencies needed to run the services.
   5. Create playbooks to deploy and manage the services on the managed nodes.

## Accumulated knowledge

### Project 1

**Deployment 3.**

- Admin Services stack (all containers)
  - Wikijs + PostgreSQL.
  - Netbox + PostgreSQL + Redis Queue + Redis Cache.
- Nginx Reverse Proxy
- Cockpit
- Podman
- Compose-Orchestrator.
  - podman-compose

### Project 2

**Deployment 1.**:

- Monitoring stack
  - Telegraf
  - InfluxDB
  - Grafana

See the Architecture design monitoring for the complete architecture of the project.

### Project 3

**Deployment 1.**:

- Domain Controller:
  - FreeIPA.

### Project 4 (Actual Guide)

**Deployment 1.**:

- Configuration Management:
  - Ansible.

## Prerequisites


