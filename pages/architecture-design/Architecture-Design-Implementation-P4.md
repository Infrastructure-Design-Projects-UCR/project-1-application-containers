
# Architecture Design Implementation - Project 4

Project 4: Configuration Management with **Ansible**

Describing how it will be implemented in the infrastructure, using the domain DNS services already configured to manage the infrastructure with **Ansible**.

# First Deployment Details

## 1. Set up Ansible in the `vm-mgmt` machine

This will be the control node for Ansible because of the domain DNS services already configured and the availability of the necessary tools.

Also because this machine already have a network with ssh permission to the other machines in the network.

Here we need to do the following steps:

1. Install the **Ansible** package on the control node with dnf.

    - This is because Rocky Linux is based on RHEL, and it have the Ansible package available in the default repositories in their most latest version.
        - The best version to install for this project is `ansible` package, because we need `ansible-galaxy` to use roles and collections from **Ansible Galaxy**.
    - Also install the needed ansible lint packages to ensure the playbooks are well written and follow best practices. Best packages:
        - `ansible-lint`
        - `yamllint`

2. Create an Ansible project directory in admin local user.
    - Will be created with git clone command to clone the Ansible project repository from GitHub.
    - This directory will contain all the Ansible playbooks, roles, and inventory files.
    - The directory can be named `ansible-project` or similar, and it should be created in the home directory of the admin user.
    - The directory structure should look like this:

        ```bash
        ~/ansible-project/
        ├── ansible.cfg
        ├── inventory/
        │   └── infra2.ini
        ├── roles/
        ├── playbooks/
        ├── group_vars/
        └── host_vars/
        ```

3. Configure the **Ansible configuration file** to set the inventory path and other necessary settings.
    - Create a file named `ansible.cfg` in the `ansible-project` directory.
    - We will modify this file to set the inventory path to the `inventory` directory in our Ansible project.
    - Set as ssh user the `sshuser` created in the previous phases of the project.
    - Set as remote user the `admin-local` user created in the previous phases of the project.
    - We can use the following configuration as a starting point:

        ```ini
        [defaults]
        inventory = ./inventory/infra2.ini
        remote_user = admin-local
        become = true
        become_method = sudo
        sshuser = sshuser
        # This is the path to the SSH private key file for the sshuser
        private_key_file = ~/.ssh/ansible_admin_ed25519
        # To enable ansible retry connection attempts
        retries = 3
        timeout = 10
        ```

4. Configure the **Ansible inventory file** to include all managed nodes.
    - As we have different VMs functionalities, we will create a structured inventory file that groups the nodes by their roles.
        - We can have groups like `[services]`(vm-wikijs-netbox, vm-monitor, vm-reproxy), `[management]`(vm-mgmt) and `[domain]`(vm-freeipa)
    - In folder `ansible-project/inventory/`, we will create a file named `infra2.ini`.

## 2. Set up **SSH key-based authentication** for passwordless access to managed nodes

This is necessary to allow Ansible to connect to the managed nodes without requiring a password each time.

1. Let's test the remote and localhost with the `ping` module to ensure Ansible can communicate with them.
    - Ensure this works for the next steps to work correctly.
2. Configure the SSH keys for the admin user on the control node.
    - Generate an SSH key pair on the control node if it doesn't already exist.
    - Ensure using ed25519 keys for better security.
    - Name the key file as `ansible_admin_ed25519`.
3. Copy the public key to each remote node's `~/.ssh/authorized_keys` file for the admin user.
    - For this use ansible module for key management.
    - Use User and Password authentication to copy the public key to the remote nodes.
    - Remember using the ssh user for this step. As it is the user that has access to the remote nodes.
    - Modify the `ansible.cfg` as needed to perform this step correctly.
4. Enable passwordless SSH access for the admin user on the control node to all managed nodes.
    - Make sure the `ansible.cfg` file is correctly configured to use the SSH user and remote user, and have passwordless SSH access set up.

## 3. Check sudo permissions for the admin user on the remote nodes

This is necessary to allow Ansible to execute commands with elevated privileges on the managed nodes.

1. Just execute a simple Ansible command to check if the admin user has sudo permissions on the remote nodes.
2. Remember add `admin-local` password. And then in a environment variable `ANSIBLE_BECOME_PASS` to avoid entering the password each time.

## 4. Create Ansible playbooks for initial package installation and configuration of the remote nodes

This is necessary to ensure that the remote nodes have the necessary packages installed and configured to run the services we need.

List of total packages to install in the remote nodes:

| Tool | Purpose |
| ---- | ---- |
| python3 | Python 3 interpreter. |
| python3-pip | Python 3 package manager. |
| podman | Container runtime (rootless mode). |
| podman-compose | Container orchestration. |
| nano | Terminal Editor. |
| nmap | Network checking. |
| curl | Transfer data from or to a server. |
| bat | Cat with syntax highlighting. |
| audit2why | SELinux troubleshooting. |
| git | Version control system. |

1. Group packages into variables for flexibility.
    - Specify versions if necessary.
2. Create Ansible playbooks to install the necessary management packages on the remote nodes.
    - This can be done using the `dnf` or `pip` modules in Ansible.

3. Also ensure the previous packages versions are the required and if not, update them to the latest version available in the repositories.

## 5. Create Ansible playbooks to check the network connectivity between required nodes

This is necessary to ensure that all nodes can communicate with each other as expected. Of course, we are assuming that the network and interfaces for Ansible connectivity are already configured and working. In this case, we assume Management network is configured previously manually and also External network is configured previously manually.

1. Create Ansible playbooks to check the three networks we have in the infrastructure:
    - External network
    - Management network
    - Services network

2. Define your network in **groups_vars** or **host_vars**, depending on your needs.

3. Use network modules for `nmcli` to see the nodes info about their network interfaces and IP addresses.
    - This will help us to ensure that the nodes are correctly configured and can communicate with each other.
4. Depending on the results of the previous step, we can create Ansible playbooks to configure the network interfaces and IP addresses of the nodes if needed.
    - Ansible can reach at IP address level only. The network interfaces have to be enabled in Hipervisor level.
    - This case would happened with **Services network**, as it is not configured yet in the hypervisor level. We can assign them the subnet, IP address and default gateway using Ansible playbooks.
    - This playbook need to confirm the connection between VMs in the Services network doing a ping test between them.
5. Create Ansible playbooks to check the DNS resolution between the nodes.
    - This playbook need to confirm the DNS resolution between VMs in the Services network to see if they can resolve each other's hostnames.
6. Create Ansible playbooks to check internet connectivity from the nodes.
    - This playbook need to confirm the internet connectivity from VMs in the Services network to see if they can reach external resources like `google.com` or `github.com`.

## 6. Create Ansible playbooks to check firewall rules and ensure they are correctly configured

This is necessary to ensure that the firewall rules are correctly configured and allow the necessary traffic between the nodes.

1. Define in variables your firewall zones and the necessary ports for each service in the infrastructure.  
2. Create Ansible playbooks to check the firewall rules on the remote nodes.
    - This can be done using the `firewalld` module in Ansible.
    - This playbook should check the firewall rules for each zone configured in the remote nodes.
3. Create Ansible playbooks to ensure that the necessary firewall rules are configured to allow traffic between the nodes.
    - This can be done using the `firewalld` module in Ansible.
    - We need to ensure that that all the network interfaces are correctly configured in the firewall zones to allow the respective traffic for domain controller services, management services, monitor services and web services.
    - So first ensure that the firewall zones are correctly configured in the remote nodes by sending commands to specific zones and ports.
    - If not, add the necessary firewall rules to allow traffic between the nodes in the respective zones.

4. Create Ansible playbooks to remove any unnecessary firewall rules that may be blocking traffic between the nodes.

## 7. Create Ansible playbooks for managing the services in the remote nodes

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

**NOTE:** As you see, we have some services that are already running in **podman-compose** and others that are running with **podman** and others running as **bare metal** service.

With that information, we have a very heterogeneous environment, so we need to create Ansible playbooks to manage each type of service accordingly.

Clarify that we are going to use `podman-compose` to manage the services that are running in `podman-compose`, so we are making use of deployment files that are already created previously in the project for those. But, this can be done with absolute ansible podman modules too, but we are going to use `podman-compose` for simplicity and consistency with the previous phases of the project.

We will use the podman module orchestrator with the `vm-monitor` because they run with simple `podman` commands.

Steps for managing the services:

1. Create Ansible playbooks to manage core web services.
    - Begin with the services running in `services` inventory group.
    - Ensure that the playbooks are idempotent, meaning they can be run multiple times without changing the state of the system if it is already in the desired state.
    - Implement tasks to manage the services running in `podman-compose`.
        - The compose files are already created in the `ansible-project` directory, so we can use them to manage the services.
    - Implement tasks with the `podman` module to manage the services running in `podman`.

2. Create Ansible playbooks to manage the monitoring services.
    - Ensure `podman` networks, volumes and secrets are correctly configured for the services.
    - Implement tasks to manage the services running in `podman`.

3. Create Ansible playbooks to manage the reverse proxy service.
    - Ensure the Nginx configuration is correctly set up to route traffic to the respective services.
    - Implement tasks to manage the configuration files and the service itself. This file should be located in the `ansible-project` directory.
    - Manage the service as a bare metal service with systemctl module.
    - Make use of handlers to reload the Nginx service when the configuration files change.

4. Create Ansible playbooks to manage monitoring agent service.
    - Ensure the Telegraf configuration is correctly set up to collect metrics from the services.
    - Implement tasks to manage the configuration files and the service itself. This file should be located in the `ansible-project` directory.
    - Manage the service as a bare metal service with systemctl module.
    - Copy the configuration file to the remote nodes using the `copy` module in Ansible.
    - Make use of handlers to restart the Telegraf service when the configuration files change.

**NOTES**

- Add the necessary roles.

tags: [wikijs, netbox, monitoring, nginx, telegraf]
