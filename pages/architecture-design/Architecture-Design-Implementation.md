
# 1. **Virtual Machine strategy**

Section to define the virtual machines that we are going to use in our infrastructure, and the purpose of each machine. Also the network interfaces and firewall rules for each machine.

## 1.1. **vm-mgmt**

### 1.1.1. Purpose

Here we have all the management tools, like **Cockpit**, **Podman** and the security tools. Also have the privileges to and security accesses to manage the other machines.

### 1.1.2. Network

- **ExtNetHome**: NAT or bridged networking.
- **IntNetMgmt**: Private network for management.

### 1.1.3. Firewall

- **ExtNetHome**: Allow SSH (port 2222) and Cockpit (9090).
- **IntNetMgmt**: Allow SSH (port 22) and Cockpit (9090).

## 1.2. **vm-reproxy**

### 1.2.1. Purpose

The `vm-reproxy` machine serves as the reverse proxy for internal and external traffic routing. It ensures proper load balancing, traffic management, and an additional security layer for exposed services. This VM will host **Nginx** as the reverse proxy solution.

### 1.2.2. Network

- **ExtNetHome**: NAT or bridged networking for external access.
- **IntNetMgmt**: Private network for internal communication.

### 1.2.3. Firewall

- **ExtNetHome**: Allow HTTP (port 80) and HTTPS (port 443) for web traffic.
- **IntNetMgmt**: Allow internal HTTP traffic for reverse proxy operations.

## 1.3. **vm-wikijs-netbox**

### 1.3.1 Purpose

Here we have the services that we want to expose to the internet. The idea is to have a reverse proxy to manage the traffic and also to have a better security posture.

### 1.3.2 Container Services

- **WikiJS**: WikiJS is a modern and powerful wiki engine that allows you to create and manage documentation, knowledge bases, and other types of content. It is built on Node.js and provides a user-friendly interface for creating and editing content.
- **NetBox**: NetBox is an open-source IP address management (IPAM) and data center infrastructure management (DCIM) tool. It is designed to help network engineers manage and document their networks, including IP addresses, devices, racks, and connections.
- **Redis**: Redis is an in-memory data structure store that can be used as a database, cache, and message broker. It is often used as a caching layer for applications to improve performance and reduce latency.
- **PostgreSQL**: PostgreSQL is a powerful, open-source relational database management system (RDBMS) that is widely used for storing and managing structured data. It is known for its robustness, scalability, and support for advanced features like transactions, indexing, and complex queries.

## 1.4. **vm-monitoring**

### 1.4.1. Purpose

The `vm-monitoring` machine will be dedicated to infrastructure monitoring, logging, and alerting. It will host tools like **Prometheus**, **Grafana**, and **Telegraf** to collect and analyze performance metrics.

### 1.4.2. Network

- **IntNetMgmt**: Private network for management and data collection.
- **IntNetSrv**: Reverse proxy network for internal services.

### 1.4.3. Firewall

- **IntNetMgmt**: Allow Telegraf (port 8125) for monitoring services.
- **IntNetSrv**: Allow Grafana (port 3000) for web access.

## 1.5. **vm-freeipa**

### 1.5.1. Purpose

The `vm-freeipa` machine will be dedicated to FreeIPA, which provides centralized authentication, authorization, and account management for the infrastructure. It will also serve as the DNS server for the domain.

---

# 2. **Deployment plan**

Incremental deployment plan, to be able to test the integration with **Podman** and the services.

## 2.1. **Project 1: Application container**

### 2.1.1. Pre-Deployment

Base image configuration already has Cockpit installed and `cockpit-podman` extension. So all the machines will have the same base image.

The rationale of having Cockpit installed is be able to see the machines and the containers running in the machines with web GUI.

### 2.1.2. First deployment

All of these will be executed as admin user, it have wheel privileges, but any command will be run as sudo.

1. We will deploy the `vm-mgmt` machine, and install all the management tools.
    1. Install **Ansible**.
    2. Configure the firewalld rules.
    3. Configure the network interfaces.
2. We will deploy the `vm-reproxy` machine, but we will not install anything in this machine.
3. We will deploy the `vm-wikijs-netbox` machine, and install all the services in this machine.
    1. Pull required images.
    2. Create virtual networks for the containers.
    3. Create volumes for the containers.
    4. Run the containers with the required parameters and secret environment variables.
    5. Use resource limits for the containers. Our lab not requires a lot of resources because we are in a lab, so we can use the following limits:
        1. WikiJS container: 1GB RAM, 2 vCPU.
        2. NetBox container: 1GB RAM, 2 vCPU.
        3. Redis container: 512MB RAM, 1 vCPU.
        4. Postgres is not listed here because it will be moved to another VM in the future.

        **NOTE**: The resource limits does **not working with rootless user**, and I will keep it like this for now, security here is better than performance limit.

        **NOTE**: Also the above limits were take it from the official documentation or references of the services, on which are the minimum requirements for the services to run.

    6. Use health checks for the containers.

**Note**: No security checking is made in the first deployment, because we are in a lab to test the integration with **Podman**.

### 2.1.3. Second deployment

Deploy all with `podman-compose`, `ansible` or other orchestration tool.
    1. Use **Ansible** to deploy all the machines and the containers.
    2. Use **podman-compose** to deploy all the containers.
    3. Use **ansible** to configure the machines and the containers.

### 2.1.4. Third deployment

Deploy in `vm-reproxy` machine with `nginx` server with container.
    1. Install **nginx** in the `vm-reproxy` machine with `podman`.
    2. Configure the firewalld rules.
    3. Configure the network interfaces.

### 2.1.5. Fourth deployment

Enchance the security of the containers and machines with **Trivy** and **linPEAS**.
    1. Remember firewalld rules for each interface.
    2. Run **Trivy** and **linPEAS** in the `vm-wikijs-netbox` machine.
    3. Modify containers listen address, to only listen to required ips and ports in groups of services.

## 2.2. **Project 2: Monitoring stack**

### 2.2.1. Pre-Deployment

1. We are going to use an extra disk for the InfluxDB data. The idea is to have a separate disk for the data, so we can manage the data separately from the OS. Insolation of date is a good practice, moreover we need more space.

    After mounting the disk we will follow this plan:

2. Create a new VM with the name `vm-monitor`.
3. Configure the network interfaces.
4. Configure the firewalld rules.

### 2.2.1. First deployment

#### 2.2.1.1. InfluxDB

1. Create a persistent volume for the data in the new disk.
2. Create a new network for all the monitoring containers.
3. Pull V3 or Core container image of InfluxDB.
4. Configure InfluxDB machine to listen to other VMs and itself.
5. Make sure that InfluxDB communication method is permitted in the firewall.

#### 2.2.1.2. Telegraf

1. Install Telegraf as package.
2. Configure Telegraf for all the machines to send the data to InfluxDB. And to collect the most important metrics from the machines.
    - System resources: CPU, memory, disk and network. Is the basic metrics that we need to monitor.
3. Manage the security of the Telegraf machine.

#### 2.2.1.3. Grafana

1. Create a persistent volume for the data in the new disk.
2. Pull the latest Grafana image.
3. Configure Grafana to read the data from InfluxDB in monitoring machine.
4. Configure Grafana basic dashboards to monitor the machines cpu, memory, disk and network usage.
    - Create a dashboard for overall usage of the machines inside the infrastructure.
    - Create a dashboard for each machine
      - `vm-mgmt`
      - `vm-reproxy`
      - `vm-wikijs-netbox`
      - `vm-monitor`
5. Configure Grafana to send alerts to the admin user in case of high usage of cpu, memory, disk or network high latency.
    - Create alerts for each machine
      - `vm-mgmt`
      - `vm-reproxy`
      - `vm-wikijs-netbox`
      - `vm-monitor`
    - Services to send alerts to consider:
      - Email
      - Discord
      - Webhook
6. Perform tests to check the alerts and the dashboards.

7. Put the **Grafana** machine behind reverse proxy.
    - Configure **Nginx** backend to listen to the Grafana machine IP and port.

8. Adjust **firewalld** rules for each interface as needed.

### 2.2.1. Second deployment

Integrate security tools in the monitoring stack and vm-monitor machine.

1. Check containers and machines with **Trivy** and **linPEAS**.
2. Check `audit2why` and activate **SELinux** enforcing.

## 2.3. **Project 3: FreeIPA**

Take in count all the implementation is following FreeIPA CLI and UI enablers.

### 2.3.1. Pre-Deployment

### 2.3.2. First Deployment

The plan of implementation is to deploy the FreeIPA server in the `vm-freeipa` machine.

1. In the `vm-freeipa` machine we will install the FreeIPA server, and configure it to be the domain controller for the infrastructure.
2. Configure the FreeIPA server to be the DNS server for the domain.
3. Configure the FreeIPA server to be the authentication server for the infrastructure with LDAP.
4. Configure client machines to use the FreeIPA server for authentication and authorization.
5. Configure sysadmins and application-support groups in FreeIPA.
6. Configure host groups in FreeIPA.
7. Configure HBAC rules in FreeIPA to allow access to the services and machines. By mapping user groups to host groups and defining access policies.
8. Configure sudo rules in FreeIPA to allow sysadmins and application-support groups to execute commands with sudo.
9. Configure SSH rules in FreeIPA to allow sysadmins and application-support groups to access the machines via SSH.

For further implementation details we have the next section based in security architecture design. With the user and host groups, domain access policies we are planning to implement: [Architecture Design Implementation - P3](Architecture-Design-Implementation-P3.md).

## 2.4. **Project 4: Configuration Management**

### 2.4.1. Pre-Deployment

Prerequisites to be able to run the Ansible playbooks and manage the infrastructure with Ansible.

1. DNS configuration is already done in the FreeIPA server. `vm-freeipa` machine.
2. VM for Ansible control node is already created as `vm-mgmt`.
3. Check VMs `SSH` access and connectivity for all machines in the infrastructure. Ansible needs SSH access to all machines to manage them.
4. Check for if `ssh` user is already created in all machines. Ansible needs a user to connect to the machines. (sshuser)
5. Check if already have a user with `sudo` privileges in all machines. Ansible needs `sudo` privileges to manage the machines for initial configuration and deployment. (admin-local)
6. Check all the packages needed in VMs to run the actual state of the infrastructure. In future steps we will use Ansible to manage the packages and services in the VMs.
7. Check `python`  in all machines. Ansible needs Python to run the playbooks and manage the machines.
8. Backup the current state of the infrastructure, so we can restore it in case of any issues during the deployment process.

### 2.4.2. First Deployment

First configuration plan of Ansible to manage the infrastructure.

1. Set up Ansible in the `vm-mgmt` machine. This will be the control node for Ansible.
2. Configure Ansible inventory file to include all managed nodes.
3. Set up **SSH key-based authentication** for passwordless access to managed nodes.
4. Create Ansible playbooks to check the network connectivity between required nodes.
5. Create Ansible playbooks to check firewall rules and ensure they are correctly configured.
6. Create Ansible playbooks for initial package installation and configuration of the remote nodes.
7. Create Ansible playbooks for managing the services in the remote nodes.

To see the complete configuration management guide, see the [Configuration Management Guide](Architecture-Design-Implementation-P4.md).

### 2.4.3. Second Deployment

Here we are going to implement advanced Ansible features to manage the infrastructure. 

1. Create Ansible roles for each service and machine.
2. Increase the performance of the Ansible playbooks.
