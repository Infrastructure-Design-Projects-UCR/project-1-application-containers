
# Architecture Design Implementation - Project 3

Project 3: FreeIPA

## First Deployment Details

### **User Groups and Host Classifications**

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
   - **dc-hosts**  
     - **vm-freeipa**: Runs FreeIPA (including the UI, DNS, etc.); should be accessible only by authorized users.

---

### **Domain access policies**

We begin this the design with not allowing any access to anything, we build the rules from the ground up, allowing only the necessary access for each user group and host.

#### **Sudo rules**

Define the sudo rules for each user group to ensure they have the necessary permissions without compromising security.

**sysadmin_sudo:**

- **Purpose**: Sudo allowance for sysadmins.
- **Rules**:
  - Allow all commands.
- **User groups**:
  - `sysadmins` group.
- **Host groups**:
  - All hosts; `app-services-hosts`, `management-hosts`, `monitoring-hosts` and `dc-hosts`.

---

**application_sudo:**

- **Purpose**: Sudo allowance for application-support group.
- **Rules**:
  - Allow specific commands related to application management.
- **User groups**:
  - `application-support` group.
- **Host groups**:
  - All application related hosts; `app-services-hosts`.

#### **HBAC(Host-Based Access Control) SSH rules**

**hbac_operation_sysadmin:**

- **Purpose**: SSH access for sysadmins in all hosts.
- **Rules**:
- Allow SSH access to all hosts.
- **User groups**:
  - **sysadmins** group.
- **Host groups**:
  - All hosts; `app-services-hosts`, `management-hosts`, `monitoring-hosts` and `dc-hosts`.
- **Services**: SSH

**hbac_operation_app:**

- **Purpose**: SSH access for application-support group.
- **Rules**:
  - Allow SSH access to application hosts (vm-wikijs-netbox, vm-reproxy).
- **User groups**:
  - **application-support** group.
- **Host groups**:
  - All application related hosts; `app-services-hosts`.
- **Services**: SSH

---
