# **Container Lab Architecture and Design Document**

## **Abstract**

This document outlines the architecture and design of a lightweight, locally hosted lab environment for experimenting with Podman, container security, and various admin tools. The lab will consist of multiple virtual machines (VMs) running on a host machine, utilizing Podman for container management and incorporating tools like Cockpit, WikiJS, and Netbox. The design emphasizes security, scalability, and ease of management.

Will be a simple design to be able to experiment with the different tools.

## **Contributors**

| Name | Role |
| :---- | :---- |
| Jorge Sagot | Lead Architect & Mantainer |
| Ariel Mora | Advisor & Contributor |

---

## **Table of Contents**

- [**Container Lab Architecture and Design Document**](#container-lab-architecture-and-design-document)
  - [**Abstract**](#abstract)
  - [**Contributors**](#contributors)
  - [**Table of Contents**](#table-of-contents)
  - [**Architecture Design Document**](#architecture-design-document)
    - [**1. Architecture Overview**](#1-architecture-overview)
      - [**1.1 Purpose**](#11-purpose)
      - [**1.2 Scope**](#12-scope)
      - [**1.4 Audience**](#14-audience)
      - [**1.5 High level design**](#15-high-level-design)
    - [**2. Infrastructure Components**](#2-infrastructure-components)
      - [**2.1 Host Machine (Your PC)**](#21-host-machine-your-pc)
      - [**2.3 Virtual Machines (5 VMs Total)**](#23-virtual-machines-5-vms-total)
    - [**3. Network Design**](#3-network-design)
      - [3.1. Network](#31-network)
      - [3.2. Firewall](#32-firewall)
      - [**3.3. Network dataflow**](#33-network-dataflow)
        - [IntNetMgmt Dataflow](#intnetmgmt-dataflow)
        - [IntNetSrv Dataflow](#intnetsrv-dataflow)
    - [**4. Toolchain**](#4-toolchain)
    - [**5. Simplified Backup Plan**](#5-simplified-backup-plan)
      - [**5.1. Database data**](#51-database-data)
      - [**5.2. Database**](#52-database)
      - [**5.3. Future Scalability** (Optional strategy)](#53-future-scalability-optional-strategy)
      - [**5.4. VirtualBox Snapshots**](#54-virtualbox-snapshots)
      - [**5.5. Rationale**](#55-rationale)
    - [**6. Security**](#6-security)
      - [**6.1. Podman Security**](#61-podman-security)
      - [**6.2. SELinux** (OPTIONAL)](#62-selinux-optional)
      - [**6.3. Firewall**](#63-firewall)
      - [**6.4. Security Tools**](#64-security-tools)
      - [**6.5. SSH**](#65-ssh)
    - [7. **Implementation strategy**](#7-implementation-strategy)

---

## **Architecture Design Document**

### **1. Architecture Overview**

#### **1.1 Purpose**

The purpose of this document is to provide a comprehensive overview of the architecture and design of a lightweight, locally hosted lab environment for experimenting with Podman, container security, observability and various admin tools.

The project will consist of multiple virtual machines (VMs) running on a host machine.

The design emphasizes security, network, implementation, and ease of management.

#### **1.2 Scope**

**Project 1**

- WikiJS, and Netbox containerized.
- podman-compose for orchestration.
- Security tools like Trivy, linPEAS and Lynis.
  - Reverse Proxy with NGINX to improve security posture.

**Objectives**

The main objectives of this project are:

- Create a lightweight lab environment for experimenting with Podman and container security.
- Deploy WikiJS and Netbox in containers.
- Implement security tools like Trivy, linPEAS, and Lynis.
- Use podman-compose for container orchestration.
- Implement a reverse proxy with NGINX to improve security posture.


**Project 2**:

- Monitoring stack with Telegraf, InfluxDB, and Grafana.

**Objectives**

The main objectives of this project are:

- Implement a monitoring stack with Telegraf, InfluxDB, and Grafana.
- Collect and visualize metrics from the other VMs.
- Set up alerts and notifications for critical events.

**Project 3**:

- Domain Controller with FreeIPA.

**Objectives**
The main objectives of this project are:

- Deploy FreeIPA as a domain controller.
- Implement Host-Based Access Control (HBAC) policies.
- Configure DNS services for the domain.
- Configure LDAP and Kerberos for authentication in servers and services.

**Project 4**:

- Configuration Management with Ansible.

**Objectives**
The main objectives of this project are:

- Implement Ansible playbooks for automating the deployment and configuration of the VMs.
- Use Ansible roles to organize tasks and manage dependencies.
- Define infrastructure desired state.
- Integrate Ansible with the monitoring stack for automated alerts and reporting.

#### **1.4 Audience**

This document is intended for they who are interested in learning about Podman, container security, observability and various admin tools. It is also intended for those who want to experiment with these tools in a lightweight lab environment in local.

---

#### **1.5 High level design**

![alt text](../../media/architecture-design/image.png)

---

### **2. Infrastructure Components**

#### **2.1 Host Machine (Your PC)**

- **Role**: Hypervisor for VMs.  
- **Tools**: VirtualBox Software Hipervisor.  
- **OS**: Windows 10.
- **Resources**: 6 CPU cores and 12 threads, 32GB RAM, 1.16 TB Storage.

#### **2.3 Virtual Machines (5 VMs Total)**

| VM Name | Purpose | OS | Resources |
| :---- | :---- | :---- | :---- |
| `vm-mgmt` | Just to manage the other VMs. Is the one we need to access with ssh to admin the others. Ansible will be here to admin | Rocky Linux 9 | 1 vCPU, 1GB RAM, 8GB Storage |
| `vm-reproxy` | Reverse Proxy (NGINX) for WikiJS/Netbox/Grafana (future public IP readiness). | Rocky Linux 9 | 1 vCPU, 2GB RAM, 8GB Storage |
| `vm-wikijs-netbox` | Host WikiJS and NetBox containers (Podman). Plus services required Redis(Podman) for Netbox, PostgreSQL(Podman) for Netbox and Wikijs | Rocky Linux 9 | 4 vCPU, 2GB RAM , 8GB Storage |
| `vm-freeipa` | Have the domain controller FreeIPA with the UI and all the components including DNS to centrally manage all the authentication | Rocky Linux 9 | 1 vCPU, 2GB RAM, 14GB Storage |
| `vm-monitor` | Host Telegraf, InfluxDB (Podman), and Grafana (Podman). | Rocky Linux 9 | 2 vCPU, 2GB RAM, 10GB Storage OS and 10GB Storage for InfluxDB data. |

---

### **3. Network Design**

#### 3.1. Network

- **Home Network as Public Network** (ExtNetHome)

Use NAT Network of VirtualBox for `vm-mgmt`, `vm-proxy` and `vm-freeipa`. This is to be able to connect to the services.

- **Private Internal Network Admin** (IntNetMgmt)

Subnet that use `vm-mgmt` to manage all the nodes through SSH and Ansible. This network is isolated from the public network to enhance security.

Also work, for **Telegraf** to collect metrics from the other VMs and send them to `vm-monitor`.

- **Private Internal Network Services** (IntNetSrv)

Subnet with the traffic of **Wikijs** and **Netbox** services, so then `vm-wikijs-netbox` and `vm-reproxy`.

Also we include here the traffic of `vm-monitor` for **Grafana** client through `vm-reproxy`.

#### 3.2. Firewall

- In `ExtNetHome`:  
  - For `vm-mgmt` allow **SSH** (port 2222).  
  - For `vm-reproxy` allow **Web Traffic** only.
  - For `vm-freeipa` allow **FreeIPA UI** (port 443) and **DNS** (port 53).
- In `IntNetMgmt`:  
  - Allow access by SSH and Telegraf metrics collection.
- In `IntNetSrv`:  
  - Allow access between services.  
  - Allow only ports WikiJS and Netbox between `vm-reproxy` and `vm-wikijs-netbox`
  - Allow access to Grafana from `vm-reproxy` to `vm-monitor`.

#### **3.3. Network dataflow**

##### IntNetMgmt Dataflow

![alt text](../../media/architecture-design/image-1.png)

##### IntNetSrv Dataflow

![alt text](../../media/architecture-design/image-2.png)

### **4\. Toolchain**

| Tool | Purpose |
| ---- | ---- |
| Ansible | Configuration management and deployment automation. |
| Podman | Container runtime (rootless mode). |
| podman-compose | Container orchestration. |
| SSH | Manage nodes manually. |
| nano | Terminal Editor. |
| nmap | Network checking. |
| Trivy/linPEAS | Container/image scanning. |
| SELinux | Mandatory access control. |
| curl | Transfer data from or to a server. |
| bat | Cat with syntax highlighting. |
| firewalld | Firewall management. |
| network-manager | Network management for Rocky Linux. |
| audit2why | SELinux troubleshooting. |

---

### **5. Simplified Backup Plan**

#### **5.1. Database data**

- **All backups run from `vm-postgres`**.  
- For data **rely on Podman volumes** for persistent storage.

#### **5.2. Database**

1. **PostgreSQL Dumps**  (OPTIONAL)
   - Run `pg_dumpall` inside the Postgres container, saving to a **host-mounted directory** (e.g., `/backups` on `vm-postgres`).  
   - Retain for 7 days (auto-purge older files).

2. **Volume Snapshots**  
   - Use `podman volume inspect` to locate data directories.  
   - Optional: Script to **tar** critical volumes (e.g., WikiJS/Netbox) if they reside on `vm-postgres`.

#### **5.3. Future Scalability** (Optional strategy)

- Add a `vm-backups` VM later for:  
  - Centralized backup storage (NFS/S3).  
  - Encryption/offsite sync (e.g., `rclone` to cloud).  
  - Ansible-driven automation.

#### **5.4. VirtualBox Snapshots**

- **VM Snapshots**: Use VirtualBox to take snapshots of VMs.

#### **5.5. Rationale**

- **No new tools**: Uses existing Podman/Postgres/VirtualBox functionalities.  
- **No extra resources**: Backups consume only disk space on `vm-postgres`.  
- **Recovery-ready**:  
  - Restore DBs via `psql` from dumps.  
  - Recreate volumes from host files.
  - Recreate machine states from VirtualBox snapshots.

---

### **6. Security**

#### **6.1. Podman Security**

First thing to declare here, is that we will run Podman in rootless mode, providing almost a null probability of privilege escalation. The attack surface is significantly reduced, and the risk of container breakout is minimized.

We will have an unprovileged user to run Podman.

#### **6.2. SELinux** (OPTIONAL)

The idea is run SELinux in enforcing mode, so we will have a better security posture. But we are in a lab to test the integration with **Podman**.

#### **6.3. Firewall**

We will use **firewalld** to manage the firewall rules. The idea is to have a minimal set of rules to allow only the traffic that we need.

Also network splits in subnets to have a better security posture. Because we will have a private network for the management and another one for the services. The idea is to have a better security posture, and also to have a better performance.

#### **6.4. Security Tools**

Using **Trivy** and **linPEAS** to scan the containers and the host machine. The idea is to have a better idea on the security of container and implement contrameasures to mitigate the risks.

Using of **Lynis** to scan the host machines. (If I have time to implement it).

#### **6.5. SSH**

**SSH** hardening by changing the default port to 2222, and also using **key-based authentication**. Also avoiding the use of password authentication and the root login.

**Ansible** and **Cockpit** will be use SSH for the communication with the nodes. So we will have to use SSH keys for that.

### 7. **Implementation strategy**

See the [Machine strategy](Architecture-Design-Implementation.md) for more details.
