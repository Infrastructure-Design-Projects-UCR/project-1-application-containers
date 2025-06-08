
# 1. **Virtual Machine strategy**

Section to define 

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

### 2.1. **Project 1: Application container**

#### 2.1.1. Pre-Deployment

Base image configuration already has Cockpit installed and `cockpit-podman` extension. So all the machines will have the same base image.

The rationale of having Cockpit installed is be able to see the machines and the containers running in the machines with web GUI.

#### 2.1.2. First deployment

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

#### 2.1.3. Second deployment

Deploy all with `podman-compose`, `ansible` or other orchestration tool.
    1. Use **Ansible** to deploy all the machines and the containers.
    2. Use **podman-compose** to deploy all the containers.
    3. Use **ansible** to configure the machines and the containers.

#### 2.1.4. Third deployment

Deploy in `vm-reproxy` machine with `nginx` server with container.
    1. Install **nginx** in the `vm-reproxy` machine with `podman`.
    2. Configure the firewalld rules.
    3. Configure the network interfaces.

#### 2.1.5. Fourth deployment

Enchance the security of the containers and machines with **Trivy** and **linPEAS**.
    1. Remember firewalld rules for each interface.
    2. Run **Trivy** and **linPEAS** in the `vm-wikijs-netbox` machine.
    3. Modify containers listen address, to only listen to required ips and ports in groups of services.

### 2.2. **Project 2: Monitoring stack**

#### 2.2.1. Pre-Deployment

1. We are going to use an extra disk for the InfluxDB data. The idea is to have a separate disk for the data, so we can manage the data separately from the OS. Insolation of date is a good practice, moreover we need more space.

    After mounting the disk we will follow this plan:

2. Create a new VM with the name `vm-monitor`.
3. Configure the network interfaces.
4. Configure the firewalld rules.

#### 2.2.1. First deployment

##### 2.2.1.1. InfluxDB

1. Create a persistent volume for the data in the new disk.
2. Create a new network for all the monitoring containers.
3. Pull V3 or Core container image of InfluxDB.
4. Configure InfluxDB machine to listen to other VMs and itself.
5. Make sure that InfluxDB communication method is permitted in the firewall.

##### 2.2.1.2. Telegraf

1. Install Telegraf as package.
2. Configure Telegraf for all the machines to send the data to InfluxDB. And to collect the most important metrics from the machines.
    - System resources: CPU, memory, disk and network. Is the basic metrics that we need to monitor.
3. Manage the security of the Telegraf machine.

##### 2.2.1.3. Grafana

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

#### 2.2.1. Second deployment

Integrate security tools in the monitoring stack and vm-monitor machine.

1. Check containers and machines with **Trivy** and **linPEAS**.
2. Check `audit2why` and activate **SELinux** enforcing.

### 2.3. **Project 3: FreeIPA**

Take in count all the implementation is following FreeIPA CLI and UI enablers.

#### 2.3.1. Pre-Deployment

#### 2.3.2. First Deployment

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

For further implementation details we have the next section based in security architecture design. With the user and host groups, domain access policies we are planning to implement:

##### **User Groups and Host Classifications**

1. **User Groups**
    a. **Infrastructure**:
      - **sysadmins:**  
        - **Purpose**: Members in this group have unrestricted **sudo** abilities and full system management rights.
        - **Rules**: They must have access to every interface and service (FreeIPA UI, Grafana, WikiJS, NetBox, and SSH on all hosts).
      - **application-support:**
        - **Purpose**: Members in this group only can execute sudo permissions on application-related commands (e.g., managing WikiJS, NetBox, Redis, and PostgreSQL).
        - **Rules**:
          - They can SSH into the application hosts (vm-wikijs-netbox, vm-reproxy) but not into management hosts (vm-monitor, vm-mgmt, vm-freeipa).
          - They should not have access to the **FreeIPA UI**.

    b. **Web UI**:
      - **freeipa-ui-access:**  
        - This group can be created to clearly delineate which users (usually overlapping with sysadmins) are allowed to interact with the FreeIPA UI.
      - **grafana-ui-access:**  
        - This group can be created to manage access to the Grafana interface.
      - **wikijs-ui-access:**  
        - This group can be created to manage access to the WikiJS interface.
      - **netbox-ui-access:**  
        - This group can be created to manage access to the NetBox interface.

2. **Host Groups**
   - **app-services-hosts**
     - **vm-wikijs-netbox**: Hosts WikiJS, NetBox, and supporting services (e.g., Redis).
     - **vm-reproxy**: Serves as the public/reverse proxy for WikiJS, NetBox, Grafana, and (if applicable) FreeIPA UI.
   - **monitoring-hosts**
     - **vm-monitor**: Dedicated to monitoring services (e.g., Prometheus, Grafana) and telemetry agents.
   - **management-hosts**
     - **vm-mgmt**: Used for administrative SSH access to manage the other VMs.
     - **vm-freeipa**: Runs FreeIPA (including the UI, DNS, etc.); should be accessible only by authorized users.

---

##### **Domain access policies**

We begin this the design with not allowing any access to anything, we build the rules from the ground up, allowing only the necessary access for each user group and host.

###### Sudo rules

Define the sudo rules for each user group to ensure they have the necessary permissions without compromising security.

**sysadmin_sudo:**

- **Purpose**: Sudo allowance for sysadmins.
- **Rules**:
  - Allow all commands.
- **User groups**:
  - **sysadmins** group.
- **Host groups**:
  - All hosts; `app-services-hosts`, `management-hosts`, and `monitoring-hosts`.

---

**application_sudo:**

- **Purpose**: Sudo allowance for application-support group.
- **Rules**:
  - Allow specific commands related to application management.
- **User groups**:
  - **application-support** group.
- **Host groups**:
  - All application related hosts; `app-services-hosts` and **Grafana** UI only.

#### SSH rules

**sysadmin_ssh:**

- **Purpose**: SSH access for sysadmins in all hosts.
- **Rules**:
- Allow SSH access to all hosts.
- **User groups**:
  - **sysadmins** group.
- **Host groups**: 
  - All hosts; `app-services-hosts`, `management-hosts`, and `monitoring-hosts`.

**application_ssh:**

- **Purpose**: SSH access for application-support group.
- **Rules**:
  - Allow SSH access to application hosts (vm-wikijs-netbox, vm-reproxy).
- **User groups**:
  - **application-support** group.
- **Host groups**:
  - All application related hosts; `app-services-hosts`.

---
