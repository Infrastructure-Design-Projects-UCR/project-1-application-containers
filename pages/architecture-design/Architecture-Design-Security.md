
# Security Architecture Design

## Overview

Security architecture design is a critical aspect

## Domain Controller Policy

For this iteration, we will focus on Host-Based Access Control (HBAC) as a key component of security architecture design. HBAC is a method of controlling access to resources based on the host from which the request originates.

It allows administrators to define policies that restrict access to services and resources based on the host's identity, enhancing security by ensuring that only authorized hosts can access specific services.

We are going to use a fine-grained approach to HBAC, adding rules for each service we have in our infrastructure, so that we can add this rules only to the hosts and users that need to access the service in granular way.

### **User Groups and Host Classifications**

1. **User Groups**
    a. **Infrastructure**:
      - **sysadmins:**  
        - Members in this group have unrestricted **sudo** abilities and full system management rights.
        - They must have access to every interface and service (FreeIPA UI, Grafana, WikiJS, NetBox, and SSH on all hosts).
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
   - **management-hosts**
     - **vm-monitor**: Hosts Grafana, InfluxDB, and telemetry agents.
     - **vm-mgmt**: Used for administrative SSH access to manage the other VMs.
     - **vm-freeipa**: Runs FreeIPA (including the UI, DNS, etc.); should be accessible only by authorized users.

---

### **Access and Service Rules**

We begin this the design with not allowing any access to anything, we build the rules from the ground up, allowing only the necessary access for each user group and host.

1. **FreeIPA Web Interface (vm-freeipa)**
   - **Allowed:**  
     - Only users in the **sysadmins** (or a designated FreeIPA UI access group) are permitted to access the FreeIPA UI.
   - **Enforcement:**  
     - Implement Host-Based Access Control (HBAC) rules within FreeIPA.

2. **Web Services via Reverse Proxy (vm-reproxy)**
   - **sysadmins:**  
     - Unrestricted access to all interfaces (FreeIPA, Grafana, WikiJS, NetBox).
   - **application-support:**  
     - Access to WikiJS and NetBox interfaces via vm-reproxy.
   - **Enforcement:**  
     - Configure NGINX on vm-reproxy to serve content based on URL paths and enforce access rules (backed by authentication/authorization via FreeIPA, if possible).

3. **SSH and Sudo Access**
   - **management-manage-rule:**
     - Full SSH access on all hosts (vm-mgmt, vm-freeipa).
     - Unrestricted sudo permissions for full administrative control.
   - **application-manage-rule:**  
     - SSH access allowed on all application hosts (vm-wikijs-netbox, vm-reproxy) but not on management hosts (vm-monitor, vm-mgmt, vm-freeipa).
     - Sudo rights are **restricted**: only allowed to run a specific set of commands (e.g., managing WikiJS, NetBox, Redis, and PostgreSQL).
   - **Enforcement:**  
     - Utilize FreeIPA sudo rules to specify allowed commands for the application-support group.
     - Apply HBAC rules as needed on each host.
   - **Note**:
      - The **sysadmins** group will have unrestricted sudo access across all hosts, so is going to have both the `management-manage-rule` and `application-manage-rule` applied to them.

---
