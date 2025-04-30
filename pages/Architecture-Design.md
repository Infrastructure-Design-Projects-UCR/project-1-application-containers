# **Container Lab Architecture and Design Document**

## **Abstract**

This document outlines the architecture and design of a lightweight, locally hosted lab environment for experimenting with Podman, container security, and various admin tools. The lab will consist of multiple virtual machines (VMs) running on a host machine, utilizing Podman for container management and incorporating tools like Cockpit, WikiJS, and Netbox. The design emphasizes security, scalability, and ease of management.

Will be a simple design to be able to experiment with the different tools.

## **Contributors**

| Name | Role |
| :---- | :---- |
| Jorge Sagot | Lead Architect & Mantainer |
| Ariel Mora | Advisor & Contributor |

## **Table of Contents**

## **Architecture Design Document**

### **1\. Architecture Overview**

**Objective**: A lightweight, locally hosted lab for experimenting with Podman, container security, and admin tools (Cockpit, WikiJS, Netbox).

#### **Logical Design**

*(Example: VMs on your PC → Podman containers \+ Reverse Proxy \+ Security Tools)*

---

### **2\. Infrastructure Components**

#### **2.1 Host Machine (Your PC)**

- **Role**: Hypervisor for VMs.  
- **Tools**: VirtualBox Software Hipervisor.  
- **OS**: Rocky Linux 9 Minimal (host OS).

#### **2.3 Virtual Machines (5 VMs Total)**

| VM Name | Purpose | OS | Resources |
| :---- | :---- | :---- | :---- |
| `vm-mgmt` | Host Cockpit (web UI), Podman, and security tools (Trivy, linPEAS). | Rocky Linux 9 | 2 vCPU, 1GB RAM, 8GB Storage |
| `vm-proxy` | Reverse Proxy (NGINX) for WikiJS/Netbox (future public IP readiness). | Rocky Linux 9 | 1 vCPU, 2GB RAM, 8GB Storage |
| `vm-wikijs-netbox` | Host WikiJS and NetBox containers (Podman). Plus services required Redis for Netbox | Rocky Linux 9 | 4 vCPU, 2GB RAM , 8GB Storage |
| `vm-postgres` | Postgresql for WikiJS and Netbox machines. | Rocky Linux 9 | 1 vCPU, 1GB RAM, 14GB Storage |
| `vm-test` | Sandbox for testing other containerized services. | Rocky Linux 9 | 1 vCPU, 1GB RAM, 8GB Storage |

**NOTE**: In the future we would have an extra VM for the database to apply replication and follow other dba best practices for High Availability and Replication.

---

### **3\. Network Design**

#### 3.1 Network

- **Home Network as Public Network** (ExtNetHome)

Use NAT or bridged networking for `vm-podman-mgmt` and `vm-proxy` VMs. This is to be able to connect to the services.

- **Private Internal Network Admin** (IntNetMgmt)

Subnet that use `vm-mgmt` to manage all the nodes.

- **Private Internal Network Services** (IntNetSrv)

Subnet with the traffic of wikijs and netbox services, so then `vm-wikijs-netbox` and `vm-proxy`.

#### 3.2 Firewall

- In `ExtNetHome`:  
  - For `vm-mgmt` allow **SSH** (port 2222), **Cockpit** (9090).  
  - For `vm-proxy` allow **Web Traffic** only.  
- In `IntNetMgmt`:  
  - Allow access by SSH.  
- In `IntNetSrv`:  
  - Allow access between services.  
  - Allow only ports 3000 (WikiJS) and 8000 (Netbox) between `vm-proxy` and `vm-wikijs-netbox`

### **4\. Toolchain**

| Tool | Purpose |
| :---- | :---- |
| Podman | Container runtime (rootless mode). |
| Cockpit | Web-based management. |
| cockpit-podman | Extension to manage containers. |
| cockpit-file | Extension to manage files. |
| Ansible | Execute instructions on remote nodes. |
| SSH | Manage nodes manually. |
| nano | Terminal Editor. |
| nmap | Network checking. |
| Trivy/linPEAS | Container/image scanning. |
| SELinux | Mandatory access control. |

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

### 7. **Machine strategy**

#### 7.1. `vm-mgmt`

##### Purpose

Here we have all the management tools, like **Cockpit**, **Podman** and the security tools. Also have the privileges to and security accesses to manage the other machines.

##### Network

- **ExtNetHome**: NAT or bridged networking.
- **IntNetMgmt**: Private network for management.

##### Firewall

- **ExtNetHome**: Allow SSH (port 2222) and Cockpit (9090).
- **IntNetMgmt**: Allow SSH (port 22) and Cockpit (9090).

#### 7.2. `vm-proxy`

#### 7.3. `vm-wikijs-netbox`

##### 7.3.1 Purpose

Here we have the services that we want to expose to the internet. The idea is to have a reverse proxy to manage the traffic and also to have a better security posture.

##### 7.3.2 Container Services

- **WikiJS**: WikiJS is a modern and powerful wiki engine that allows you to create and manage documentation, knowledge bases, and other types of content. It is built on Node.js and provides a user-friendly interface for creating and editing content.
- **NetBox**: NetBox is an open-source IP address management (IPAM) and data center infrastructure management (DCIM) tool. It is designed to help network engineers manage and document their networks, including IP addresses, devices, racks, and connections.
- **Redis**: Redis is an in-memory data structure store that can be used as a database, cache, and message broker. It is often used as a caching layer for applications to improve performance and reduce latency.
- **PostgreSQL**: PostgreSQL is a powerful, open-source relational database management system (RDBMS) that is widely used for storing and managing structured data. It is known for its robustness, scalability, and support for advanced features like transactions, indexing, and complex queries.

#### 7.4 **Deployment plan**

Incremental deployment plan, to be able to test the integration with **Podman** and the services.

##### Pre-Deployment

Base image configuration already has Cockpit installed and `cockpit-podman` extension. So all the machines will have the same base image.

The rationale of having Cockpit installed is be able to see the machines and the containers running in the machines with web GUI.

##### First deployment

All of these will be executed as admin user, it have wheel privileges, but any command will be run as sudo.

1. We will deploy the `vm-mgmt` machine, and install all the management tools.
    1. Install **Trivy** and **linPEAS**.
    2. Install **Ansible**.
    3. Configure the firewalld rules.
    4. Configure the network interfaces.
2. We will deploy the `vm-proxy` machine, but we will not install anything in this machine.
3. We will deploy the `vm-wikijs-netbox` machine, and install all the services in this machine.
    1. Pull required images.
    2. Create virtual networks for the containers.
    3. Create volumes for the containers.
    4. Run the containers with the required parameters and secret environment variables.

**Note**: No security checking is made in the first deployment, because we are in a lab to test the integration with **Podman**.

##### Second deployment

1. We will deploy all the same but with a user with limited privileges to run Podman. The above user was `wheel` user, so it creates all the containers. But anyways the above deployment was with `sudo` privileges.
2. Enchance the security of the containers with **Trivy** and **linPEAS** in the `vm-wikijs-netbox` machine.
    1. Remember firewalld rules for each interface.
    2. Run **Trivy** and **linPEAS** in the `vm-wikijs-netbox` machine.
    3. Modify containers listen address, to only listen to required ips and ports in groups of services.
3. Create set of services as `pods`, so we can manage all the containers as a single units.

#### 7.5. Services configuration

##### 7.4.1 **Planning**

###### **On container privilages**

**This lab:**

- By this lab we run all as rootful user, to test the integration with **Podman**.

**Future improvements**

- In the future we will run all as rootless user, to have a better security posture.

###### **On container execution**

**This lab:**

- By this lab we run each service in a different container, and with podman common CLI.

**Future improvements**

- In future version we will run with `podman-compose`, `ansible` or other orchestration tool.

##### 7.4.3 **WikiJS**

**Image**

We need to get the WikiJS image from the official WikiJS repository of a stable version. The idea is maintain this version image.

**Requirements**