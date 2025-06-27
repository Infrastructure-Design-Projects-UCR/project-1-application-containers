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

# Steps

## Ansible Installation and Configuration

1. Install Ansible on the control node.

    ```bash
    sudo dnf install ansible
    ```

2. Verify the Ansible installation.

    ```bash
    ansible --version
    ```

3. Now we are going to install needed collections from Ansible Galaxy.

    ```bash
    ansible-galaxy collection install ansible.posix
    ansible-galaxy collection install community.general
    ansible-galaxy collection install containers.podman
    ```

4. We are going to use a github repo to store our Ansible project files. So we need to clone the repository to the control node.

    ```bash
   git clone https://github.com/Infrastructure-Design-Projects-UCR/ansible-project.git
    ```

5. Change to the project directory:

    ```bash
    cd ansible-project
    ```

6. Create a `requirements.txt` file in the root of the project directory with the following content:

    ```text
    # requirements.txt
    ansible-lint
    yamllint
    ```
    
    **NOTE**: Install pip if needed.

7. Create a virtual environment to isolate the Ansible project dependencies:

    ```bash
    python3 -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt
    ```

8. Now create the `ansible.cfg` file in the root of the project directory with the following content:

    ```ini
    [defaults]
    # This is the default inventory file for Ansible.
    inventory = ./inventory/infra2.ini
    # Explicit remote user to use.
    remote_user = admin-local
    remote_port = 22

    # This disables host key checking, which is useful for testing but not recommended for production.
    host_key_checking = false

    # This is the path to the SSH private key file for the sshuser
    #private_key_file = ~/.ssh/ansible_admin_ed25519
    private_key_file = ~/.ssh/ansible_admin_ed25519

    # To enable ansible retry connection attempts
    retries = 3
    timeout = 10

    # Number of parallel forks to use for running tasks.
    forks = 10

    # Ansible logs
    log_path = ./ansible.log

    [privilege_escalation]
    #  This is the default connection type to use.
    become = false
    become_method = sudo
    become_user = admin-local
    ```

9. And now make the inventory file in the `inventory` directory, named `infra2.ini`, with the following content:

    ```ini
    [all_remote_nodes:children]
    services
    domain

    [services:children]
    container
    bare_metal

    [container]
    vm-wikijs-netbox
    vm-monitor

    [bare_metal]
    vm-reproxy

    [domain]
    vm-freeipa

    [controller]
    vm-mgmt

    [edge_nodes:children]
    bare_metal
    domain
    controller
    ```

## Set up SSH key-based authentication

1. Let's test the remote and localhost with the `ping` module to ensure Ansible can communicate with them.

    ```bash
    ansible all -m ping --ask-pass
    ```

    - We need to receive a `pong` response from each node to confirm connectivity.

2. Configure the SSH keys for the admin user on the control node.
    - Generate an SSH key pair on the control node if it doesn't already exist.

        ```bash
        ssh-keygen -t ed25519 -f ~/.ssh/ansible_admin_ed25519 -C "ansible_ssh_key"
        ```

    - This will create a private key (`ansible_admin_ed25519`) and a public key (`ansible_admin_ed25519.pub`) in the `~/.ssh/` directory.

3. Copy the public key to each remote node's `~/.ssh/authorized_keys` file for the admin user.

    - Now share the public key with the remote nodes with `ansible.posix.authorized_key` module.

        ```bash
        ---
        - name: Set authorized key taken from file
          hosts: all_remote_nodes
          tasks:
            - name: Add SSH public key to authorized_keys
              ansible.posix.authorized_key:
                user: "{{ ansible_user }}"
                state: present
                key: "{{ lookup('file', '~/.ssh/ansible_admin_ed25519.pub') }}"
        ```

    - Execute the playbook to copy the public key to the remote nodes:

        ```bash
        ansible-playbook playbooks/sshkey_propagate.yml --ask-pass
        ```

    - If you use the following command to test the playbook:

        ```bash
        ssh -i ~/.ssh/ansible_admin_ed25519 admin-local@vm-reproxy
        ```

        You should be able to log in without entering a password.

    - Now test the passwordless SSH access by running the following command:

        ```bash
        ansible all -m ping
        ```

## Test become privileges

To test the `become` privileges, you can run the following command:

```bash
ansible all_remote_nodes -m command -a "whoami" --become 
```

- If everything is set up correctly, you should see the output showing the user as `admin-local` on each node.

## Create a Snapshot in VirtualBox to save the current state of all nodes

This is necessary to ensure that we can revert to a known good state if something goes wrong during the configuration process.

Just go to the VirtualBox GUI, select each VM, and create a snapshot with a descriptive name like `Ansible Configuration - Initial State`.

## Create Ansible playbooks for initial package installation and configuration of the remote nodes

This is necessary to ensure that the remote nodes have the necessary packages installed and configured to run the services we need.

List of total packages to install in the remote nodes:

1. Group packages into variables for flexibility.
    - Specify versions if necessary.

2. Create Ansible playbooks to install the necessary management packages on the remote nodes.
    - This can be done using the `dnf` or `pip` modules in Ansible.

3. Also ensure the previous packages versions are the required and if not, update them to the latest version available in the repositories.

See the packages that we are going to install in the remote nodes:

file: `ansible-project/inventory/group_vars/all/packages.yml`

```yaml
dnf_common_packages:
  # Common packages for all systems
  - name: epel-release
  - name: python3
  - name: python3-pip
  - name: nano
  - name: nmap
  - name: curl
  - name: bat
  - name: git
  # security tools
  - name: policycoreutils-python-utils
  - name: fail2ban
  - name: firewalld

pip_common_packages:
  # Common Python packages for all systems
  - name: placeholder
```

file: `ansible-project/inventory/group_vars/container/packages.yml`

```yaml
dnf_container_packages:
  - name: podman
    version: 5.4.0

pip_container_packages:
  - name: podman-compose
    version: 1.4.0
```

See the playbook `ansible-project/inventory/group_vars/all/packages.yml` that cover all of this:

```yaml
--- # Play to install DNF and PIP packages
# For all the machines
- name: Install common packages
  hosts: controller #edge_nodes
  become: true
  gather_facts: false
  tasks:
    - name: Install common DNF packages
      ansible.builtin.dnf:
        name: >
          {{ item.name }}{{ (':' + item.version) if item.version is defined else '' }}
        state: present
      loop: "{{ dnf_common_packages | default([]) }}"
      become: true
      become_user: root

    - name: Install common PIP packages
      ansible.builtin.pip:
        name: "{{ item.name }}"
        version: "{{ item.version | default('latest') }}"
        state: present
      loop: "{{ pip_common_packages | default([]) }}"

# Container specific packages
- name: Install container-specific packages
  hosts: container
  become: true
  gather_facts: false
  tags:
    - skip_container_packages
  tasks:
    - name: Install container DNF packages
      ansible.builtin.dnf:
        name: >
          {{ item.name }}{{ (':' + item.version) if item.version is defined else '' }}
        state: present
      loop: "{{ dnf_container_packages | default([]) }}"
      become: true
      become_user: root


    - name: Install container PIP packages
      ansible.builtin.pip:
        name: "{{ item.name }}"
        version: "{{ item.version | default('latest') }}"
        state: present
      loop: "{{ pip_container_packages | default([]) }}"
```

---

## Create Ansible playbooks to check the network connectivity between required nodes


This is necessary to ensure that all nodes can communicate with each other as expected.

Of course, we are assuming that the network and interfaces for Ansible connectivity are already configured and working, so the `NetIntMgmt` interface and subnet is properly set up.

Also we already have set up the Domain Service network with the FreeIPA server, so we can assume that the `NetIntDomain` interface and subnet is also properly set up.

In this case, we assume Management network is configured previously manually and also External network is configured previously manually.

1. Create Ansible playbooks to check the four networks we have in the infrastructure: 
    - External network
    - Domain network
    - Management network
    - Services network

2. Define your network in **host_vars**, depending on your needs. We save in our host_vars the network interfaces and subnets for each node in the infrastructure. Also we have here the firewall configuration. 

    See this example of the `ansible-project/inventory/host_vars/vm-mgmt.yml` file:

    ```yaml
    network_interfaces:
    - conn_name: ExtNetHome
        ifname: enp0s3
        type: ethernet
        method4: manual
        ip4:
        - 10.0.3.10/24
        gw4: 10.0.3.1
        route_metric4: 100
        method6: disabled

    - conn_name: IntNetMgmt
        ifname: enp0s8
        type: ethernet
        method4: manual
        ip4:
        - 192.168.200.10/24
        route_metric4: 102
        method6: disabled

    - conn_name: IntNetDomain
        ifname: enp0s9
        type: ethernet
        method4: manual
        ip4:
        - 192.168.202.10/24
        gw4: 192.168.202.1
        dns4:
        - 192.168.202.11
        route_metric4: 101
        method6: disabled

    firewalld_config:
    default_zone: public
    version: "1.3.4"
    zones:
        - name: dmz
        forward: true
        forward_ports: []
        icmp_block_inversion: false
        icmp_blocks: []
        interfaces:
            - enp0s8
        masquerade: false
        ports: []
        protocols: []
        rich_rules: []
        services:
            - ssh
        source_ports: []
        sources: []
        target: default

        - name: public
        forward: true
        forward_ports: []
        icmp_block_inversion: false
        icmp_blocks: []
        interfaces:
            - enp0s3
            - enp0s9
        masquerade: false
        ports: []
        protocols: []
        rich_rules: []
        services:
            - ssh
            - http
            - https
            - dns
        source_ports: []
        sources: []
        target: default
    ```

3. Then we run the following playbook `network_management.yml` to check the network connectivity between the nodes:

    ```yaml
    ---
    - name: Network Management Playbook
    hosts: all
    become: true
    become_user: root
    gather_facts: false
    tasks:
        - name: Configure network interfaces
        community.general.nmcli:
            conn_name: "{{ item.conn_name }}"
            ifname: "{{ item.ifname }}"
            type: "{{ item.type }}"
            ip4: "{{ item.ip4 }}"
            gw4: "{{ item.gw4 | default(omit) }}"
            dns4: "{{ item.dns4 | default(omit) }}"
            route_metric4: "{{ item.route_metric4 | default(omit) }}"
            method4: "{{ item.method4 | default('manual') }}"
            method6: "{{ item.method6 | default('disabled') }}"
            state: present
        loop: "{{ network_interfaces }}"
        notify: Reload NetworkManager configuration

        #- name: Up the network interfaces
        #community.general.nmcli:
        #  conn_name: "{{ item.conn_name }}"
        #  state: up
        #loop: "{{ network_interfaces | default([]) }}"
        #when: item.method4 != 'disabled'

    handlers:
        - name: Reload NetworkManager configuration
        ansible.builtin.command: nmcli connection reload
        changed_when: false

    ```

## Create Ansible playbooks to check firewall rules and ensure they are correctly configured

This is necessary to ensure that the firewall rules are correctly configured and allow the necessary traffic between the nodes.

1. In the same way as before we use the **hosts_vars** to config our firewall rules with the followings playbooks:

Playbook: `ansible-project/playbooks/firewalld_management.yml`

```yaml
---
- name: Firewalld Management Playbook
  hosts: all
  become: true
  become_user: root
  gather_facts: false
  vars:
    zones_list: "{{ firewalld_config.zones }}"
  tasks:
    - name: Ensure firewalld service is running and enabled
      ansible.builtin.service:
        name: firewalld
        state: started
        enabled: true

    - name: Configure each firewalld zone
      ansible.builtin.include_tasks: firewalld_configs.yml
      loop: "{{ zones_list }}"
      loop_control:
        loop_var: zone
        label: "{{ zone.name }}"
```

and the `firewalld_configs.yml` file:

```yaml
---
- name: Ensure zone existing zone {{ zone.name }}
  ansible.posix.firewalld:
    zone: "{{ zone.name }}"
    state: present
    permanent: true

- name: Add interfaces to {{ zone.name }}
  ansible.posix.firewalld:
    zone: "{{ zone.name }}"
    interface: "{{ item }}"
    state: enabled
    immediate: true
    permanent: true
  loop: "{{ zone.interfaces }}"
  when: zone.interfaces | length > 0

- name: Add services to {{ zone.name }}
  ansible.posix.firewalld:
    zone: "{{ zone.name }}"
    service: "{{ item }}"
    state: enabled
    immediate: true
    permanent: true
  loop: "{{ zone.services }}"
  when: zone.services | length > 0

- name: Add ports to {{ zone.name }}
  ansible.posix.firewalld:
    zone: "{{ zone.name }}"
    port: "{{ item }}"
    state: enabled
    immediate: true
    permanent: true
  loop: "{{ zone.ports }}"
  when: zone.ports | length > 0

- name: Add source ip addresses to {{ zone.name }}
  ansible.posix.firewalld:
    zone: "{{ zone.name }}"
    source: "{{ item }}"
    state: enabled
    immediate: true
    permanent: true
  loop: "{{ zone.sources }}"
  when: zone.sources | length > 0

- name: Add protocols to {{ zone.name }}
  ansible.posix.firewalld:
    zone: "{{ zone.name }}"
    protocol: "{{ item }}"
    state: enabled
    immediate: true
    permanent: true
  loop: "{{ zone.protocols }}"
  when: zone.protocols | length > 0

- name: Enable masquerade on {{ zone.name }}
  ansible.posix.firewalld:
    zone: "{{ zone.name }}"
    masquerade: "{{ zone.masquerade }}"
    state: enabled
    immediate: true
    permanent: true

- name: Set target for {{ zone.name }}
  ansible.posix.firewalld:
    zone: "{{ zone.name }}"
    target: "{{ zone.target }}"
    state: enabled
    immediate: false
    permanent: true

- name: Toggle ICMP block inversion on {{ zone.name }}
  ansible.posix.firewalld:
    zone: "{{ zone.name }}"
    icmp_block_inversion: "{{ zone.icmp_block_inversion }}"
    state: enabled
    immediate: true
    permanent: true
```

## Create Ansible playbooks for managing the services in the remote nodes

This is necessary to ensure that the services are correctly configured and running on the remote nodes.

We have the following services to manage in the remote nodes already configured in the previous phases of the project:

| VM | Service | Purpose |
| --- | --- | ---- |
| vm-wikijs-netbox | Wikijs(podman-compose) | Wiki service. |
| vm-wikijs-netbox | PostgreSQL for Wikijs(podman-compose) | Database service for Wikijs. |
| vm-wikijs-netbox | Netbox(podman-compose) | Network management service. |
| vm-wikijs-netbox | PostgreSQL for Netbox(podman-compose) | Database service for Netbox. |
| vm-wikijs-netbox | Redis Queue for Netbox(podman-compose) | Queue service for Netbox. |
| vm-wikijs-netbox | Redis Cache for Netbox(podman-compose) | Cache service for Netbox. |
| vm-reproxy | Nginx | Reverse proxy service. |
| vm-monitor | Telegraf| Monitoring agent service. |
| vm-monitor | InfluxDB(podman) | Time-series database service. |
| vm-monitor | Grafana(podman) | Monitoring dashboard service. |

Steps for managing the services:

### Create Ansible playbooks to manage core web services

1. First to implement a play to manage the services running in `podman-compose` we need to ensure that the `podman` module is installed in the control node.

2. First we need to setup volumes, networks and secrets needed for the services running in `podman-compose`. This step is made with the `podman` collection in Ansible.

    - Use vault to encrypt the following secret created in previous steps of the project with the services environment variables:

        ```bash
        ansible-vault encrypt /files/services/secrets/netbox-secrets.yml
        ansible-vault encrypt /files/services/secrets/wikijs-secrets.yml
        ```

    - Those files follow this format:

        ```yaml
        # netbox_secrets.yml (encrypted with Ansible Vault)
        netbox_set_redis_cache_password_secret: "placeholder"
        netbox_set_superuser_email_secret: "placeholder"
        netbox_set_postgres_password_secret: "placeholder"
        netbox_set_postgres_user_secret: "placeholder"
        netbox_set_redis_task_password_secret: "placeholder"
        netbox_set_superuser_pass_secret: "placeholder"
        netbox_set_superuser_name_secret: "placeholder"
        netbox_set_postgres_db_secret: "placeholder"
        netbox_set_secret_key_secret: "placeholder"
        netbox_set_auth_ldap_bind_password_secret: "placeholder"
        ```

        ```yaml
        # wikijs_secrets.yml (encrypted)
        wikijs_set_postgres_db_secret: "placeholder"
        wikijs_set_postgres_user_secret: "placeholder"
        wikijs_set_postgres_password_secret: "placeholder"
        wikijs_set_wikijs_db_type_secret: "placeholder"
        wikijs_set_wikijs_db_host_secret: "placeholder"
        wikijs_set_wikijs_db_port_secret: "placeholder"
        ```

    - Then we can use a playbook to setup all the volumes, networks and secrets needed for the services running in `podman-compose`. The file `ansible-project/playbooks/podman_setup_wikijs_netbox.yml` will contain the necessary tasks.

        ```yaml
        ---
        - name: Podman environment setup for vm-wikijs-netbox
        hosts: vm-wikijs-netbox
        gather_facts: false
        vars_files:
            - ../files/services/secrets/netbox-secrets.yml  # Encrypted with Ansible Vault
            - ../files/services/secrets/wikijs-secrets.yml  # Encrypted with Ansible Vault
        vars:
            podman_networks:
            - name: wikijs-service-set
                net: "10.89.0.0/24"
                gateway: "10.89.0.1"
            - name: netbox-service-set
                net: "10.89.1.0/24"
                gateway: "10.89.1.1"

            # Volumes for persistent data, they use default local user directory, not bind mounts
            podman_volumes:
            - name: wikijs-set-postgres-data
            - name: netbox-set-media-data
            - name: netbox-set-postgres-data
            - name: netbox-set-redis-cache-data
            - name: netbox-set-redis-task-queue-data
            - name: netbox-set-reports-data
            - name: netbox-set-scripts-data

            podman_secrets:
            # Secrets for Wikijs
            - name: wikijs_set_postgres_db
                content: "{{ wikijs_set_postgres_db_secret }}"
            - name: wikijs_set_postgres_user
                content: "{{ wikijs_set_postgres_user_secret }}"
            - name: wikijs_set_postgres_password
                content: "{{ wikijs_set_postgres_password_secret }}"
            - name: wikijs_set_wikijs_db_type
                content: "{{ wikijs_set_wikijs_db_type_secret }}"
            - name: wikijs_set_wikijs_db_host
                content: "{{ wikijs_set_wikijs_db_host_secret }}"
            - name: wikijs_set_wikijs_db_port
                content: "{{ wikijs_set_wikijs_db_port_secret }}"

            # Secrets for Netbox
            - name: netbox_set_redis_cache_password
                content: "{{ netbox_set_redis_cache_password_secret }}"
            - name: netbox_set_superuser_email
                content: "{{ netbox_set_superuser_email_secret }}"
            - name: netbox_set_postgres_password
                content: "{{ netbox_set_postgres_password_secret }}"
            - name: netbox_set_postgres_user
                content: "{{ netbox_set_postgres_user_secret }}"
            - name: netbox_set_redis_task_password
                content: "{{ netbox_set_redis_task_password_secret }}"
            - name: netbox_set_superuser_pass
                content: "{{ netbox_set_superuser_pass_secret }}"
            - name: netbox_set_superuser_name
                content: "{{ netbox_set_superuser_name_secret }}"
            - name: netbox_set_postgres_db
                content: "{{ netbox_set_postgres_db_secret }}"
            - name: netbox_set_secret_key
                content: "{{ netbox_set_secret_key_secret }}"
            - name: netbox_set_auth_ldap_bind_password
                content: "{{ netbox_set_auth_ldap_bind_password_secret }}"

        tasks:
            #- name: Ensure Podman is installed TODO
            #  ansible.builtin.package:
            #    name: podman
            #    state: present
            #  become: true
            #  become_user: root

            #- name: Ensure podman-compose is installed TODO

            - name: Create Podman networks
            containers.podman.podman_network:
                name: "{{ item.name }}"
                subnet: "{{ item.net | default(omit) }}"
                gateway: "{{ item.gateway | default(omit) }}"
                state: present
            loop: "{{ podman_networks }}"

            - name: Create Podman volumes
            containers.podman.podman_volume:
                name: "{{ item.name }}"
                state: present
            loop: "{{ podman_volumes }}"

            - name: Create Podman secrets
            containers.podman.podman_secret:
                name: "{{ item.name }}"
                data: "{{ item.content | default(omit) }}"
                state: present
                skip_existing: true
            loop: "{{ podman_secrets }}"

        ```

3. Now we need to use common `shell` or `command` modules to run the `podman-compose` command. **THERE IS NOT PODMAN_COMPOSE MODULE**

    - The following playbook show how to execute the `podman-compose` commands to manage the services running in `podman-compose`.
    
    - First we need to set secrets environment variables and compose-files in our `ansible-project`. We have to create the following directory structure:

        ```bash
        ansible-project/
        ├── files/
        │   └── services/
        │       ├── compose/
        │       │   ├── netbox-set-compose.yml
        │       │   └── wikijs-set-compose.yml
        │       └── secrets/
        │           ├── netbox-secrets.yml
        │           └── wikijs-secrets.yml
        ```

    - We have shown the content of the `secrets/`. Now we need to create the `compose/` directory with the `netbox-set-compose.yml` and `wikijs-set-compose.yml` files.

        - `netbox-set-compose.yml`:

        ```yaml
        version: '3.8'
        services:
        postgres:
            container_name: netbox-set-postgres-service
            image: postgres:17-alpine
            networks:
            netbox-service-set:
                ipv4_address: 10.89.1.31
            ports:
            - "5433:5432"
            volumes:
            - netbox-set-postgres-data:/var/lib/postgresql/data:Z
            secrets:
            - source: netbox_set_postgres_db
                target: POSTGRES_DB
                type: env
            - source: netbox_set_postgres_password
                target: POSTGRES_PASSWORD
                type: env
            - source: netbox_set_postgres_user
                target: POSTGRES_USER
                type: env
            restart: unless-stopped

        redis-tasks:
            container_name: netbox-set-redis-tasks-service
            image: valkey:8.0-alpine
            networks:
            netbox-service-set:
                ipv4_address: 10.89.1.32
            volumes:
            - netbox-set-redis-task-queue-data:/data:Z
            command: ["valkey-server", "--appendonly", "yes"]
            restart: unless-stopped

        redis-cache:
            container_name: netbox-set-redis-cache-service
            image: valkey:8.0-alpine
            networks:
            netbox-service-set:
                ipv4_address: 10.89.1.33
            command: ["valkey-server"]
            restart: unless-stopped

        netbox:
            container_name: netbox-set-netbox-service
            image: netbox:latest-3.2.0
            networks:
            netbox-service-set:
                ipv4_address: 10.89.1.30
            ports:
            - "8000:8080"
            volumes:
            - netbox-set-media-data:/opt/netbox/netbox/media:Z
            - netbox-set-reports-data:/opt/netbox/netbox/reports:Z
            - netbox-set-scripts-data:/opt/netbox/netbox/scripts:Z
            environment:
            - ALLOWED_HOSTS=*
            - DB_WAIT_DEBUG=1
            - DB_HOST=10.89.1.31
            - DB_PORT=5432
            - REDIS_HOST=10.89.1.32
            - REDIS_PORT=6379
            - REDIS_DATABASE=0
            - REDIS_SSL=False
            - REDIS_CACHE_HOST=10.89.1.33
            - REDIS_CACHE_PORT=6379
            - REDIS_CACHE_DATABASE=1
            - REDIS_CACHE_SSL=False
            - SKIP_SUPERUSER=false
            - REMOTE_AUTH_ENABLED=True
            - REMOTE_AUTH_BACKEND=netbox.authentication.LDAPBackend
            - AUTH_LDAP_SERVER_URI=ldap://vm-freeipa.homelabdomain.lan:389
            - AUTH_LDAP_BIND_DN=uid=ldapauth-netbox,cn=users,cn=accounts,dc=homelabdomain,dc=lan
            #- AUTH_LDAP_BIND_PASSWORD=pass
            - AUTH_LDAP_USER_SEARCH_BASEDN=cn=users,cn=accounts,dc=homelabdomain,dc=lan
            - AUTH_LDAP_GROUP_SEARCH_BASEDN=cn=groups,cn=accounts,dc=homelabdomain,dc=lan
            #- AUTH_LDAP_REQUIRE_GROUP_DN=cn=groups,cn=accounts,dc=homelabdomain,dc=lan
            #- AUTH_LDAP_IS_ADMIN_DN=cn=netbox-web-admins,cn=groups,cn=accounts,dc=homelabdomain,dc=lan
            #- AUTH_LDAP_IS_SUPERUSER_DN=cn=netbox-web-superusers,cn=groups,cn=accounts,dc=homelabdomain,dc=lan
            - AUTH_LDAP_USER_SEARCH_ATTR=uid
            #- AUTH_LDAP_GROUP_SEARCH_CLASS=groupOfNames
            - AUTH_LDAP_GROUP_TYPE=GroupOfNamesType
            - AUTH_LDAP_ATTR_LASTNAME=sn
            - AUTH_LDAP_ATTR_FIRSTNAME=givenName
            - LDAP_IGNORE_CERT_ERRORS=True
            secrets:
            - source: netbox_set_postgres_user
                target: DB_USER
                type: env
            - source: netbox_set_postgres_db
                target: DB_NAME
                type: env
            - source: netbox_set_postgres_password
                target: DB_PASSWORD
                type: env
            - source: netbox_set_superuser_name
                target: SUPERUSER_NAME
                type: env
            - source: netbox_set_superuser_email
                target: SUPERUSER_EMAIL
                type: env
            - source: netbox_set_superuser_pass
                target: SUPERUSER_PASSWORD
                type: env
            - source: netbox_set_secret_key
                target: SECRET_KEY
                type: env
            - source: netbox_set_auth_ldap_bind_password
                target: AUTH_LDAP_BIND_PASSWORD
                type: env
            healthcheck:
            test: ["CMD", "curl", "-f", "http://localhost:8080/login/"]
            start_period: 90s
            interval: 15s
            timeout: 3s
            retries: 3
            restart: unless-stopped

        networks:
        netbox-service-set:
            external: true

        volumes:
        netbox-set-postgres-data:
            external: true
        netbox-set-redis-task-queue-data:
            external: true
        netbox-set-media-data:
            external: true
        netbox-set-reports-data:
            external: true
        netbox-set-scripts-data:
            external: true

        secrets:
        netbox_set_postgres_db:
            external: true
        netbox_set_postgres_password:
            external: true
        netbox_set_postgres_user:
            external: true
        netbox_set_superuser_name:
            external: true
        netbox_set_superuser_email:
            external: true
        netbox_set_superuser_pass:
            external: true
        netbox_set_secret_key:
            external: true
        netbox_set_auth_ldap_bind_password:
            external: true
        ```

        - `wikijs-set-compose.yml`:

        ```yaml
        version: '3.8'
        services:
        postgres:
            image: postgres:17-alpine
            container_name: wikijs-set-postgres-service
            networks:
            wikijs-service-set:
                ipv4_address: 10.89.0.31
            ports:
            - "5432:5432"
            volumes:
            - wikijs-set-postgres-data:/var/lib/postgresql/data:Z
            secrets:
            - source: wikijs_set_postgres_user
                target: POSTGRES_USER
                type: env
            - source: wikijs_set_postgres_password
                target: POSTGRES_PASSWORD
                type: env
            - source: wikijs_set_postgres_db
                target: POSTGRES_DB
                type: env
            restart: unless-stopped

        wikijs:
            image: wiki:2.5
            container_name: wikijs-set-wikijs-service
            depends_on:
            - postgres
            networks:
            wikijs-service-set:
                ipv4_address: 10.89.0.30
            ports:
            - "8080:3000"
            secrets:
            - source: wikijs_set_wikijs_db_type
                target: DB_TYPE
                type: env
            - source: wikijs_set_wikijs_db_host
                target: DB_HOST
                type: env
            - source: wikijs_set_wikijs_db_port
                target: DB_PORT
                type: env
            - source: wikijs_set_postgres_user
                target: DB_USER
                type: env
            - source: wikijs_set_postgres_password
                target: DB_PASS
                type: env
            - source: wikijs_set_postgres_db
                target: DB_NAME
                type: env
            healthcheck:
            test: ["CMD", "curl", "-f", "http://localhost:3000/healthz"]
            start_period: 30s
            interval: 30s
            timeout: 5s
            retries: 3
            restart: unless-stopped

        networks:
        wikijs-service-set:
            external: true

        volumes:
        wikijs-set-postgres-data:
            external: true

        secrets:
        wikijs_set_postgres_user:
            external: true
        wikijs_set_postgres_password:
            external: true
        wikijs_set_postgres_db:
            external: true
        wikijs_set_wikijs_db_type:
            external: true
        wikijs_set_wikijs_db_host:
            external: true
        wikijs_set_wikijs_db_port:
            external: true
        ```

        - Now we can execute the file `ansible-project/playbooks\deploy_manager_wikijs_netbox.yml`:

        ```yaml
        ---
        - name: Copy compose files to remote
        hosts: vm-wikijs-netbox
        become: true
        gather_facts: false
        tags:
            - up
            - down
        tasks:
            - name: Ensure compose directory exists
            ansible.builtin.file:
                path: "~/compose"
                state: directory
                mode: '0755'

            - name: Copy podman-compose YAMLs to remote compose directory
            ansible.builtin.copy:
                src: ../files/services/compose/
                dest: "~/compose/"
                mode: '0644'
                owner: "{{ ansible_user }}"
                group: "{{ ansible_user }}"

        - name: Start container services using podman-compose
        hosts: vm-wikijs-netbox
        become: true
        gather_facts: false
        tags:
            - up
            - start
        tasks:
            - name: Start NetBox stack
            ansible.builtin.command:
                cmd: podman-compose -f netbox-set-compose.yml -p netbox-set up -d
                chdir: ~/compose
            tags:
                - up
                - start

            - name: Start WikiJS stack
            ansible.builtin.command:
                cmd: podman-compose -f wikijs-set-compose.yml -p wikijs-set up -d
                chdir: ~/compose
            tags:
                - up
                - start

        - name: Stop container services using podman-compose
        hosts: vm-wikijs-netbox
        become: true
        gather_facts: false
        tags:
            - down
            - stop
        tasks:
            - name: Stop NetBox stack
            ansible.builtin.command:
                cmd: podman-compose -f ~/compose/netbox-set-compose.yml -p netbox-set down
                chdir: ~/compose
            tags:
                - down
                - stop

            - name: Stop WikiJS stack
            ansible.builtin.command:
                cmd: podman-compose -f ~/compose/wikijs-set-compose.yml -p wikijs-set down
                chdir: ~/compose
            tags:
                - down
                - stop
        ```

    The previous steps meet the dependencies of this file.

4. With that created, then we need to ensure at this point that containers are running.

   - Make sure you install the required collection from Ansible Galaxy at the beginning of this guide.
   - Use the following add-hoc command to ensure that containers and pods are well.

        ```bash
        ansible vm-wikijs-netbox -m ansible.builtin.command -a "cmd=podman ps -a"
        ```

     - You should see the output with the containers running in the `vm-wikijs-netbox` node.

        ```text
        (.venv) [admin-local@vm-mgmt ansible-project]$     ansible vm-wikijs-netbox -m ansible.builtin.command -a "podman ps -a" --ask-vault-pass
        Vault password:
        vm-wikijs-netbox | CHANGED | rc=0 >>
        CONTAINER ID  IMAGE                                          COMMAND               CREATED         STATUS                   PORTS                             NAMES
        c511a0134090  docker.io/library/postgres:17-alpine           postgres              22 minutes ago  Up 22 minutes            0.0.0.0:5433->5432/tcp            netbox-set-postgres-service
        f8d62300a1b6  docker.io/valkey/valkey:8.0-alpine             valkey-server --a...  22 minutes ago  Up 22 minutes            6379/tcp                          netbox-set-redis-tasks-service
        82c4e89aafe0  docker.io/valkey/valkey:8.0-alpine             valkey-server         22 minutes ago  Up 22 minutes            6379/tcp                          netbox-set-redis-cache-service
        05fd091f9033  docker.io/netboxcommunity/netbox:latest-3.2.0  /opt/netbox/docke...  22 minutes ago  Up 22 minutes (healthy)  0.0.0.0:8000->8080/tcp            netbox-set-netbox-service
        3d5e99c6646d  docker.io/library/postgres:17-alpine           postgres              22 minutes ago  Up 22 minutes            0.0.0.0:5432->5432/tcp            wikijs-set-postgres-service
        c3c65ae3a2e2  ghcr.io/requarks/wiki:2.5                      node server           22 minutes ago  Up 22 minutes (healthy)  0.0.0.0:8080->3000/tcp, 3443/tcp  wikijs-set-wikijs-service
        ```

---

### Create Ansible playbooks to manage the monitoring services

Ensure `podman` networks, volumes and secrets are correctly configured for the services.

This time we are going to use the `podman` collection in Ansible to manage the services running in `podman`. And orchestrate the services using `podman` commands.

Also we are going to make a full playbook to manage the monitoring stack, including **InfluxDB3** and **Grafana**, that are deployed in `podman` containers.

TODO: Guide In Progress.

```yaml

```

**Telegraf** will run on all hosts as daemon service, so we are going to create in next steps a playbook to manage the **Telegraf** service as a bare metal service for all hosts.

### Create Ansible playbooks to manage the reverse proxy service

- Ensure the Nginx configuration is correctly set up to route traffic to the respective services.
- Implement tasks to manage the configuration files and the service itself. This file should be located in the `ansible-project` directory.
- Manage the service as a bare metal service with systemctl module.
- Make use of handlers to reload the Nginx service when the configuration files change.

TODO: Guide In Progress.

### Create Ansible playbooks to manage monitoring agent service

First ensure the Telegraf configuration is correctly set up to collect metrics from the services.

1. We need to create a secret file for the **Telegraf** service configuration. **Telegraf** needs **InfluxDB3** token to write metrics to the database, so we need to create a secret file with the token and encrypt it with Ansible Vault.

    ```bash
    ansible-vault encrypt inventory/group_vars/all/influxdb_token_secret.yml
    ```

    - This file should contain the following variables:

        ```yaml
        influxdb3_token: "placeholder"
        ```

2. Also we need to have a **group_vars** file for all hosts to define the Telegraf configuration variables, such as the InfluxDB URL, token, and organization.

    - And the general information would be in the `ansible-project/inventory/group_vars/all/telegraf_service_conf.yml` file:

        ```yaml
        influxdb3_url: "http://vm-monitor.homelabdomain.lan:8181"
        influxdb3_org: "Infrastructure 2"
        influxdb3_bucket: "telegraf"
        ```

3. We use the following file: `files\services\config\telegraf_service.conf.j2`. To manage the **Telegraf** service configuration as a `Jinja2` template.

    ```toml
    # Telegraf Configuration File
    # This is a custom configuration for Telegraf monitoring agent

    # Global agent settings (we set the defaults here)
    # [agent]
    #   interval = "10s"
    #   round_interval = true
    #   metric_batch_size = 1000
    #   metric_buffer_limit = 10000
    #   collection_jitter = "0s"
    #   flush_interval = "10s"
    #   flush_jitter = "0s"
    #   precision = ""
    #   hostname = ""
    #   omit_hostname = false

    #########################
    ## Input configuration ##
    #########################

    # Read metrics about cpu usage
    [[inputs.cpu]]
    ## Whether to report per-cpu stats or not
    percpu = false
    ## Whether to report total system cpu stats or not
    totalcpu = true
    ## If true, collect raw CPU time metrics
    collect_cpu_time = false
    ## If true, compute and report the sum of all non-idle CPU states
    ## NOTE: The resulting 'time_active' field INCLUDES 'iowait'!
    report_active = true
    ## If true and the info is available then add core_id and physical_id tags
    core_tags = false

    # Memory usage metrics
    [[inputs.mem]]

    # Disk usage metrics
    [[inputs.disk]]
    ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs"]

    # Disk I/O metrics
    [[inputs.diskio]]

    # System load metrics
    [[inputs.system]]

    # Network interface metrics
    [[inputs.net]]

    # Network protocol metrics
    [[inputs.netstat]]

    # Process metrics
    [[inputs.processes]]

    # System uptime
    [[inputs.kernel]]

    # Podman metrics
    # [[inputs.docker]]
    #   endpoint = "unix:///var/run/docker.sock"
    #   gather_services = false
    #   container_names = []
    #   source_tag = false
    #   container_name_include = []
    #   container_name_exclude = []
    #   timeout = "5s"
    #   perdevice = true
    #   total = false

    # Systemd service metrics
    [[inputs.systemd_units]]
    unittype = "service"


    ##########################
    ## Output configuration ##
    ##########################

    ## InfluxDB output configuration
    #[[outputs.influxdb_v2]]
    #  urls = ["http://localhost:8181"]
    #  token = "${INFLUXDB_ADMIN_TOKEN}"
    #  organization = "Infrastructure 2"
    #  bucket = "telegraf"


    # Output plugin to send metrics to InfluxDB
    [[outputs.influxdb_v2]]
    urls = ["{{ influxdb3_url }}"]
    token = "{{ influxdb3_token }}"
    organization = "{{ influxdb3_org }}"
    bucket = "{{ influxdb3_bucket }}"
    ```

4. Now with that created, we can create a playbook to manage the Telegraf service on all hosts.

    - So let's create a playbook to manage the Telegraf service on all hosts:

In path `ansible-project/playbooks/deploy_monitor_stack_agent.yml` we have:

```yaml
---
- name: Install and configure Telegraf
  hosts: vm-freeipa,vm-reproxy,vm-mgmt
  become: true
  become_user: root
  vars:
    influxdata_repo_url: "https://repos.influxdata.com/stable/x86_64/main"
    influxdata_gpgkey_url: "https://repos.influxdata.com/influxdata-archive_compat.key"
    telegraf_config_file: "/etc/telegraf/telegraf.d/mytelegraf.conf"
    telegraf_user: "root"
    telegraf_group: "root"

  tasks:
    - name: Configure InfluxData YUM repo
      ansible.builtin.yum_repository:
        name: influxdata
        description: InfluxData Repository - Stable
        baseurl: "{{ influxdata_repo_url }}"
        enabled: true
        gpgcheck: true
        gpgkey: "{{ influxdata_gpgkey_url }}"

    - name: Ensure repo is reachable
      ansible.builtin.uri:
        url: "{{ influxdata_repo_url }}"
        method: GET
        return_content: false
      register: repo_check
      failed_when: repo_check.status != 200
      retries: 3
      delay: 5

    - name: Install telegraf package
      ansible.builtin.dnf:
        name: telegraf
        state: present

    - name: Ensure telegraf.d config directory exists
      ansible.builtin.file:
        path: /etc/telegraf/telegraf.d
        state: directory
        owner: "{{ telegraf_user }}"
        group: "{{ telegraf_group }}"
        mode: '0755'

    - name: Deploy Telegraf configuration using Jinja2 template
      ansible.builtin.template:
        src: "../files/services/config/telegraf_service.conf.j2"
        dest: "{{ telegraf_config_file }}"
        owner: "{{ telegraf_user }}"
        group: "{{ telegraf_group }}"
        mode: '0644'

    - name: Reload systemd daemon
      ansible.builtin.systemd:
        daemon_reload: true

    - name: Restart and enable telegraf service
      ansible.builtin.systemd:
        name: telegraf
        state: restarted
        enabled: true

```

---

# Learnings

Next time we configure Ansible and a central point of management as FreeIPA, use same network subnet for both. Because it can cause issues regarding the IPs you want to use for specific services.

In this case we want **Telegraf** to be on the same subnet (192.168.200.0/24), that is the **IntNetMgmt**. But we cant use that subnet easily with **Ansible** deployment of **Telegraf** because **FreeIPA DNS** configuration was made in a different subnet **IntNetDomain** (192.168.202.0/24).

I find this issue because I was trying to deploy **Telegraf** in the **IntNetMgmt** subnet, but it was not possible because the **FreeIPA DNS** configuration was made in a different subnet. This caused issues with the IPs I wanted to use for specific services, as they were not reachable due to the subnet mismatch.

We are good only because **IntNetMgmt** is connected with all the other networks as well as **IntNetDomain**.

