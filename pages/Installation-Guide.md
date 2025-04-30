# Installation guide

Project 1: Deployment 1.

## Introduction

- [Installation guide](#installation-guide)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
    - [Software requirements](#software-requirements)
    - [Network requirements](#network-requirements)
    - [Hardware requirements](#hardware-requirements)
  - [Base image](#base-image)
  - [Software install Cockpit adn addons](#software-install-cockpit-adn-addons)
    - [Cockpit](#cockpit)
      - [Cockpit podman](#cockpit-podman)
      - [Cockpit File manager](#cockpit-file-manager)
    - [Hardening the base image](#hardening-the-base-image)
      - [Install Lynis from CISOfy and use it for hardening](#install-lynis-from-cisofy-and-use-it-for-hardening)
      - [SSH](#ssh)
      - [Firewall enable only](#firewall-enable-only)
      - [Disable compilers](#disable-compilers)
      - [Put banners](#put-banners)
      - [User admin strong password](#user-admin-strong-password)
      - [Lynus enable only for admin user](#lynus-enable-only-for-admin-user)
      - [Remove all the Lynis logs](#remove-all-the-lynis-logs)
      - [History](#history)
  - [VMs setup for the lab](#vms-setup-for-the-lab)
    - [vm-mgmt](#vm-mgmt)
      - [Oracle VirtualBox Network settings:](#oracle-virtualbox-network-settings)
      - [Installation steps](#installation-steps)
        - [Ansible](#ansible)
      - [Security](#security)
      - [SSH](#ssh-1)
        - [sshuser disable bash and python](#sshuser-disable-bash-and-python)
    - [vm-wikijs-netbox](#vm-wikijs-netbox)
      - [Oracle VirtualBox Network settings](#oracle-virtualbox-network-settings-1)
      - [WikiJS service set](#wikijs-service-set)
        - [PostgreSQL](#postgresql)
        - [WikiJS](#wikijs)
        - [VirtualBox Backup](#virtualbox-backup)
      - [Netbox service set](#netbox-service-set)
        - [PostgreSQL](#postgresql-1)
        - [Redis](#redis)
        - [Netbox](#netbox)
    - [vm-reproxy](#vm-reproxy)
      - [Oracle VirtualBox Network settings](#oracle-virtualbox-network-settings-2)
    - [Cockpit web console](#cockpit-web-console)
  - [TIPS](#tips)
    - [Oracle VirtualBox](#oracle-virtualbox)
      - [See vms IP addresses](#see-vms-ip-addresses)

## Prerequisites

### Software requirements

- Oracle VirtualBox installed in the host machine.
- VirtualBox Guest Additions installed in the guest machine.

### Network requirements

- Create a **Host-Only** network in VirtualBox for `IntNetSrv` the VMs to communicate with each other.
- Create a **Host-Only** network in VirtualBox for `IntNetMgmt` the VMs to communicate with each other.

**Reference link**: [VirtualBox Host-Only Network](https://www.youtube.com/watch?v=cxCvv__AfCY)

### Hardware requirements

- Memory: 8GB of RAM in the host machine.
- CPU: 4 cores in the host machine.
- Disk: 50GB of disk space in the host machine.

## Base image

guide[admin@rocky-test-image ~]$ history
    1  ls
    2  ll
    3  hostname
    4  ping google.com
    5  sudo dnf install nano
    6  nmtui
    7  sudo nmtui
    8  sudo reboot
    9  ip a
   10  yum
   11  yum --version
   12  dnf --version
   13  sudo yum nmap
   14  sudo yum install nmap
   15  time
   16  date
   17  sudo dnf install nmap
   18  ping google.com
   19  sudo nano /etc/resolv.conf
   20  sudo dnf install nmap
   21  sudo dnf -t upgrade
   22  sudo dnf upgrade
   23  sudo reboot
   24  sudo dnf install gcc kernel-devel kernel-headers make bzip2 perl
   25  reboot
   26  sudo reboot
   27  cd /run/medi
   28  cd /run/
   29  ll
   30  cd media
   31  ll
   32  cd ..
   33  cd media/
   34  ll
   35  ls
   36  ll -a
   37  cd
   38  ll
   39  ll -a
   40  cd /run/mount/
   41  ll
   42  ll -a
   43  reboot
   44  sudo reboot
   45  /opt/
   46  ll
   47  ll -a
   48  mount
   49  mount -l
   50  ll
   51  ls
   52  getent

Deactivate SELinux enforcing mode:

   53  sestatus
   54  sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/'
   55  sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
   56  head /etc/selinux/config
   57  cat /etc/selinux/config
   58  sudo reboot

VirtualBox Guest Additions installation:
[Reference link used:](https://linuxconfig.org/install-virtualbox-guest-additions-on-linux-guest)
   59  ll
   60  cd ..
   61  ll
   62  cd ..
   63  ll
   64  cd media/
   65  ll
   66  cd ..
   67  ll -a opt/
   68  ll -a media/
   69  ll -a mnt/
   70  ll -a usr/
   71  ll -a usr/share/
   72  cd
   73  sudo mkdir -p /mnt/cdrom
   74  cd /mnt/cdrom/
   75  ll
   76  sudo mount /dev/cdrom /mnt/cdrom/
   77  ll
   78  cd /mnt/cdrom/
   79  cd /dev/cdrom
   80  cd /dev/cdrom
   81  ll /dev/
   82  ll /dev/cdrom
   83  ll
   84  pwd
   85  sudo mount /dev/cdrom /mnt/cdrom/
   86  cd /dev/cdrom
   87  pwd
   88  ll
   89  cd 
   90  ll
   91  cd /dev/cdrom
   92  ll /dev/cdrom
   93  ll /dev/
   94  ll | grep cd
   95  ll /dev | grep cd
   96  cd /mnt/cdrom/
   97  ll
   98  sudo bash autorun.sh
   99  bash autorun.sh
  100  sudo sh autorun.sh
  101  pwd
  102  sudo ./autorun.sh
  103  sudo ./VBoxLinuxAdditions.run
  104  ./VBoxLinuxAdditions.run
  105  sudo dnf install tar bzip2 kernel-devel kernel-headers perl make elfutils-libelf-devel
  106  pwd
  107  sudo ./VBoxLinuxAdditions.run
  108  sudo shutdown
  109  ll
  110  history


## Software install Cockpit adn addons

### Cockpit

1. Install Cockpit

    If the web console is not installed by default on your installation variant, manually install the cockpit package:

    ```bash
    sudo dnf install cockpit
    ```

2. Enable and start the cockpit.socket service, which runs a web server:

    ```bash
    sudo systemctl enable --now cockpit.socket
    ```

#### Cockpit podman

This addon install the `cockpit-podman` addon to manage the containers in the VMs and also install `podman` in the VM.

```bash
sudo dnf install cockpit-podman
```

Also need to enable the podman socket, in `cockpit-podman` menu in **cockpit**, we can start the podman socket in `Services > podman`.

Your web gui have to look like this:

![alt text](../media/Installation-Guide/cockpit-podman-running.png)

#### Cockpit File manager

This addon install the `cockpit-files` addon to manage the files in the web console.

```bash
sudo dnf install cockpit-files
```

### Hardening the base image

- Pre-hardening level: 67
- Post-hardening level: 76

#### Install Lynis from CISOfy and use it for hardening

1. Create the repository in `/etc/yum.repos.d/cisofy-lynis.repo`:

    ```bash
    sudo tee /etc/yum.repos.d/cisofy-lynis.repo <<'EOF'
    [lynis]
    name=CISOfy Software - Lynis package
    baseurl=https://packages.cisofy.com/community/lynis/rpm/
    enabled=1
    gpgkey=https://packages.cisofy.com/keys/cisofy-software-rpms-public.key
    gpgcheck=1
    priority=2
    ```

2. Install Lynis:

    ```bash
    sudo dnf install lynis
    ```

3. Use of Lynis to increase base image hardening.

- Audit command:

```bash
sudo lynis audit system | sudo tee /var/log/lynis-report-prehardening-$(date +%d%m%Y).log
```

#### SSH

1. SSHD configuration Lynis reference.
2. Create a `sshuser` with less privileges.
3. Config in `/etc/ssh/sshd_config.d/sshd.conf`:

```bash
"AllowTcpForwarding" "no" 
"ClientAliveCountMax" "2" 
"LogLevel" "VERBOSE" 
"MaxAuthTries" "3" 
"MaxSessions" "3" 
"TCPKeepAlive" "no" 
"X11Forwarding" "no" 
"AllowAgentForwarding" "no" 
```

<!-- 
1. Allow SSH access to `sshuser` only.

    ```bash
    sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config && echo "AllowUsers sshuser" | sudo tee -a /etc/ssh/sshd_config
    ``` 
-->

#### Firewall enable only

Enable ssh port and cockpit port for `public` firewalld zone.

#### Disable compilers

REF Lynis: HRDN-7222.

- After searching for the compilers installed in the system, we can remove them.

```bash
sudo dnf remove gcc make perl 
```

#### Put banners

Reference Lynis: HRDN-7126 and HRDN-7130.

1. And /etc/issue and /etc/issue.net:

    ```bash
    sudo tee /etc/issue /etc/issue.net <<'EOF'
    *********************************************************************
    WARNING: UNAUTHORIZED ACCESS PROHIBITED

    This system is:
    - Restricted to authorized users only
    - Actively monitored and audited 24/7
    - Subject to legal enforcement

    By accessing this system, you acknowledge:
    1. You are an authorized user
    2. All activity is logged (including keystrokes/commands)
    3. No expectation of privacy exists
    4. Violations will be reported to law enforcement

    Unauthorized use may result in civil/criminal penalties.
    *********************************************************************
    EOF
    ```

#### User admin strong password

Just use a strong password for the `admin` user.

#### Lynus enable only for admin user

In case of any attacker, it cannot execute a scan to see the hardening level of the system and other vulnerabilities.

```bash
sudo chmod o-x $(which lynis)
```

#### Remove all the Lynis logs

To prevent an attacker from seeing the hardening level of the system, we need to remove all the Lynis logs.

```bash
sudo rm -r /var/log/lynis*
```

#### History

Disabling the history of base image and hardening.
    
```bash
history -c
```

## VMs setup for the lab


### vm-mgmt

**Purpose:**
This VM es used to manage the other VMs in the lab. It is used to run the following tools:

- Host Cockpit (web UI) with Podman and File manager addons.
- Podman client.
- Security tools (Trivy, linPEAS).
- Ansible(maybe)
- Of course, SSH client.

#### Oracle VirtualBox Network settings:

Adapter 1: NAT (for internet access) with port forwarding for SSH (2222:2222) and cockpit (9090:9090).

- This is for the `ExtNetHome` network in the diagram.

Adapter 2: **Internal Network** adapter for communication with the other VMs.

- This is for the `IntNetMgmt` network in the diagram, to manage the other VMs.

#### Installation steps

##### Ansible


#### Security

#### SSH

I put banners for SSH and login.

1. SSH banner:

    ```bash
    sudo nano /etc/ssh/sshd_banner
    *****************************************************************
    WARNING: UNAUTHORIZED ACCESS PROHIBITED

    You are accessing a private computer system. This system is:
    - Restricted to authorized users ONLY
    - Actively monitored 24/7
    - Subject to audit logging and legal enforcement

    By proceeding, you:
    1. Acknowledge you are an authorized user
    2. Consent to ALL activity being monitored and recorded
    3. Accept legal liability for unauthorized actions

    All access attempts are logged with:
    - Timestamps
    - Source IP
    - Commands executed

    Violators will be prosecuted to the fullest extent of the law.
    *****************************************************************
    ```

    ```bash
    sudo sed -i 's/^#Banner.*/Banner \/etc\/ssh\/sshd_banner/' /etc/ssh/sshd_config
    ```

2. Only let `sshuser` to log in with SSH:

    ```bash
    sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config && echo "AllowUsers sshuser" | sudo tee -a /etc/ssh/sshd_config
    ```

##### sshuser disable bash and python

References: [text](https://linuxtldr.com/restricted-bash-shell/) [RBash](https://www.tecmint.com/rbash-restricted-bash-shell/)

1. We need to restrict the `sshuser` to use only a restricted shell. We can use `rbash` for this. All of these is made in the `sshuser` bash profile by admin user.

    ```bash
    sudo -u sshuser nano /home/sshuser/.bashrc
    ```

2. Put the following lines in the `.bashrc` file:

    ```bash
    # .bashrc

    # Source global definitions
    if [ -f /etc/bashrc ]; then
            . /etc/bashrc
    fi

    # User specific environment
    if ! [[ "$PATH" =~ "$HOME/.local/bin:$HOME/bin:" ]]
    then
        PATH="$HOME/.local/bin:$HOME/bin:$PATH"
    fi
    export PATH

    # Uncomment the following line if you don't like systemctl's auto-paging feature:
    # export SYSTEMD_PAGER=

    # User specific aliases and functions
    if [ -d ~/.bashrc.d ]; then
            for rc in ~/.bashrc.d/*; do
                    if [ -f "$rc" ]; then
                            . "$rc"
                    fi
            done
    fi

    unset rc

    # Force restricted shell
    set -r

    # Force su - admin command
    function su_admin() {
        if [[ "$1" == "-" && "$2" == "admin" ]]; then
            su - admin  # Only allow "su - user"
        else
            echo "Error: Only 'su - admin' is permitted."
            return 1
        fi
    }

    alias su='su_admin'
    alias ls='echo "Command blocked"'
    alias cat='echo "Command blocked"'
    alias cp='echo "Command blocked"'
    alias mv='echo "Command blocked"'
    alias rm='echo "Command blocked"'
    alias vi='echo "Command blocked"'
    alias vim='echo "Command blocked"'
    alias python='echo "Command blocked"'
    alias python3='echo "Command blocked"'
    alias ssh='echo "Command blocked"'
    alias curl='echo "Command blocked"'
    alias wget='echo "Command blocked"'
    alias exec='echo "Command blocked"'
    alias eval='echo "Command blocked"'
    alias source='echo "Command blocked"'
    alias nmap='echo "Command blocked"'
    alias ping='echo "Command blocked"'
    alias ip='echo "Command blocked"'
    # Lock down PATH
    export PATH=/usr/bin:/bin
    ```

3. Then save and exit the file. 

4. Now lockdown write permissions for the `sshuser` to bash profile files:

    ```bash
    sudo chattr +i /home/sshuser/.bashrc
    sudo chattr +i /home/sshuser/.bash_profile
    ```

- Previous steps deactivates all commands unless the `su - admin`. And the use of the `bash -r` to be able of log in with ssh and change to `admin` user.

### vm-wikijs-netbox

**Purpose**:
This VM is used to run the web applications WikiJS and Netbox. It is used to run the following tools:

- Podman (for running the web applications)
- WikiJS
- Netbox
- PostgreSQL (for WikiJS and Netbox)
- Redis (for Netbox)

#### Oracle VirtualBox Network settings

Adapter 1: **NAT Network** deactivated, only in case of test internet access if not NAT is provided by the `vm-mgmt` VM.

Adapter 2: **Internal Network** adapter for communication with the other VMs.

- This is for the `IntNetMgmt` network in the diagram, to be manage by **vm-mgmt**.

Adapter 3: **Internal Network** adapter for communication with the applications.

- This is for the `IntNetSrv` network in the diagram, to communication with **vm-reproxy** to receive requests.

#### WikiJS service set

This is a combination of two services: WikiJS and PostgreSQL.

1. Here create a network only for this set of services:

    ```bash
    podman network create wikijs-service-set
    ```

    **NOTE:** see the configuration with in `podman network inspect wikijs-service-set`.

    ```json
    "name": "wikijs-service-set",
    "id": "f224b61a04d31984cbe881353815d6a6deee2561ee35ec49bb8c634e75c52c69",
    "driver": "bridge",
    "network_interface": "podman1",
    "created": "2025-04-26T10:43:46.57167705-06:00",
    "subnets": [
      {
        "subnet": "10.89.0.0/24",
        "gateway": "10.89.0.1"
      }
    ],
    "ipv6_enabled": false,
    "internal": false,
    "dns_enabled": true,
    "ipam_options": {
      "driver": "host-local"
    },
    ```

2. Next we are going to create **PostgreSQL** things and then we are going to create the **WikiJS** service.

**NOTE**: for future deployments, we can use ENV variables with ansible, bash or python scripts. With those we can store all the needed variables as secrets and have a more secure and reusable deployment.

Also the use of `podman-compose` or podman pods can be interesting for future deployments.

##### PostgreSQL

1. Use `postgres` image to install **PostgreSQL**:

    ```bash
    podman pull postgres:17-alpine
    ```

2. Create the persistent volume for the **PostgreSQL** database data:

    ```bash
    podman volume create wikijs-set-postgres-data
    ```

3. Create the secrets for **Environment variables** with podman:

    ```bash
    printf "your_secure_password" | podman secret create wikijs_set_postgres_password -
    printf "wikijs" | podman secret create wikijs_set_postgres_user -
    printf "wikidb" | podman secret create wikijs_set_postgres_db -
    ```

4. Deploy the PostgreSQL container:

    - With secrets:

        ```bash
        podman run --name wikijs-set-postgres-service \
            -d \
            --network wikijs-service-set \
            --ip 10.89.0.31 \
            -p 5432:5432 \
            -v wikijs-set-postgres-data:/var/lib/postgresql/data:Z \
            --secret wikijs_set_postgres_user,type=env,target=POSTGRES_USER \
            --secret wikijs_set_postgres_password,type=env,target=POSTGRES_PASSWORD \
            --secret wikijs_set_postgres_db,type=env,target=POSTGRES_DB \
            --restart=unless-stopped \
            postgres:17-alpine
        ```

        **NOTE**: for future deployments, we can use ENV variables with ansible, bash or python scripts. With those we can store all the needed variables as secrets and have a more secure and reusable deployment.

    - Optionally test without secrets:

        ```bash
        podman run --name wikijs-set-postgres-service \
          -d \
          --network wikijs-service-set \
          --ip 10.89.0.31 \  # This IP was planned in the network configuration.
          -p 5432:5432 \
          -v wikijs-set-postgres-data:/var/lib/postgresql/data:Z \
          -e POSTGRES_USER=wikijs \
          -e POSTGRES_PASSWORD=your_secure_password \
          -e POSTGRES_DB=wikidb \
          --restart=unless-stopped \
          postgres:17-alpine
        ```

5. Check the PostgreSQL container is running on designed ports and IP:

    ```bash
    podman ps -a
    ```

6. Check the PostgreSQL logs:

    ```bash
    podman logs wikijs-set-postgres-service
    ```

7. Check the PostgreSQL database:

    ```bash
    podman exec -it wikijs-set-postgres-service psql -U wikijs -d wikidb
    ```

    - In postgres prompt:

    ```bash
    wikidb=# \l
    ```

##### WikiJS

1. Use `ghcr.io/requarks/wiki` to install WikiJS:

    ```bash
    podman pull ghcr.io/requarks/wiki:2.5
    ```

    **NOTE**: recommended not use `latest` tag, because it is not stable and can break the application.

2. Create the environment variables for the WikiJS container:

    - Required variables:

        ```text
        DB_TYPE=postgres
        DB_HOST=localhost
        DB_PORT=5432
        DB_USER=wikijs
        DB_PASS=pass
        DB_NAME=wikidb
        ```

    - Create the needed secrets for **Environment variables** with podman:

        ```bash
        printf "postgres" | podman secret create wikijs_set_wikijs_db_type -
        printf "IP" | podman secret create wikijs_set_wikijs_db_host -
        printf "5432" | podman secret create wikijs_set_wikijs_db_port -
        ```

    - We gonna use podman secrets for the postgres user, password and dbname created before in the PostgreSQL section.

        ```bash
        wikijs_set_postgres_password
        wikijs_set_postgres_user
        wikijs_set_postgres_db
        ```

3. Deploy the WikiJS container:

    We need to run the container with the following:
    - ip of it network
    - listen to the right WikiJS port
    - use secrets
    - set the health check to check if the service is running

    ```bash
    podman run --name wikijs-set-wikijs-service \
        -d \
        --network wikijs-service-set \
        --ip 10.89.0.30 \
        -p 8080:3000 \
        --secret wikijs_set_wikijs_db_type,type=env,target=DB_TYPE \
        --secret wikijs_set_wikijs_db_host,type=env,target=DB_HOST \
        --secret wikijs_set_wikijs_db_port,type=env,target=DB_PORT \
        --secret wikijs_set_postgres_user,type=env,target=DB_USER \
        --secret wikijs_set_postgres_password,type=env,target=DB_PASS \
        --secret wikijs_set_postgres_db,type=env,target=DB_NAME \
        --health-cmd="curl -f http://localhost:3000/healthz || exit 1" \
        --health-start-period=30s \
        --health-retries=3 \
        --restart=unless-stopped \
        wiki:2.5
    ```

    **NOTE**: for future deployments, we can use ENV variables with ansible, bash or python scripts. With those we can store all the needed variables as secrets and have a more secure and reusable deployment.

    - Optionally test without secrets:

    ```bash
    podman run --name wikijs-set-wikijs-service \
        -d \
        --network wikijs-service-set \
        --ip 10.89.0.30 \
        -p 8080:3000 \
        -e DB_TYPE=postgres \
        -e DB_HOST=localhost \
        -e DB_PORT=5432 \
        -e DB_USER=wikijs \
        -e DB_PASS=pass \
        -e DB_NAME=wikidb \
        --restart=unless-stopped \
        wiki:2.5
    ```

4. Check the WikiJS container:

    ```bash
    podman ps -a
    ```

5. Check the WikiJS logs:

    ```bash
    podman logs wikijs-set-wikijs-service
    ```

##### VirtualBox Backup

Take a snapshot of the VM `vm-wikijs-netbox` with the PostgreSQL and WikiJS services running.

#### Netbox service set

This is a combination of three services: Netbox, PostgreSQL and Redis.

1. First, we need to define the network for the **Netbox** service:

    ```bash
    podman network create netbox-service-set
    ```

    **NOTE:** see the configuration with in `podman network inspect netbox-service-set`.

      ```json
      [
        {
          "name": "netbox-service-set",
          "id": "4297740540cc0422c7929151a31fa5092c25b12e83d5203240b8b285ae5f549f",
          "driver": "bridge",
          "network_interface": "podman2",
          "created": "2025-04-27T10:03:02.884937426-06:00",
          "subnets": [
            {
              "subnet": "10.89.1.0/24",
              "gateway": "10.89.1.1"
            }
          ],
          "ipv6_enabled": false,
          "internal": false,
          "dns_enabled": true,
          "ipam_options": {
            "driver": "host-local"
          },
          "containers": {}
        }
      ]
      ```

2. Then we are going to create **PostgreSQL** and **Redis** things and then we'll the **Netbox** service with podman.

##### PostgreSQL

1. Create the persistent volume for the **PostgreSQL** database data:

    ```bash
    podman volume create netbox-set-postgres-data
    ```

2. Create the secrets for **Environment variables** with podman:

    ```bash
    printf "your_secure_password" | podman secret create netbox_set_postgres_password -
    printf "netbox" | podman secret create netbox_set_postgres_user -
    printf "netboxdb" | podman secret create netbox_set_postgres_db -
    ```

3. Deploy the PostgreSQL container:

    - With secrets:

        ```bash
        podman run --name netbox-set-postgres-service \
            -d \
            --network netbox-service-set \
            --ip 10.89.1.31 \
            -p 5433:5432 \
            -v netbox-set-postgres-data:/var/lib/postgresql/data:Z \
            --secret netbox_set_postgres_db,type=env,target=POSTGRES_DB \
            --secret netbox_set_postgres_password,type=env,target=POSTGRES_PASSWORD \
            --secret netbox_set_postgres_user,type=env,target=POSTGRES_USER \
            --restart=unless-stopped \
            postgres:17-alpine
        ```

        **NOTE**: for future deployments, we can use ENV variables with ansible, bash or python scripts. With those we can store all the needed variables as secrets and have a more secure and reusable deployment.

    - Optionally test without secrets:

        ```bash
        podman run --name netbox-set-postgres-service \
            -d \
            --network netbox-service-set \
            --ip 10.89.1.31 \
            -p 5433:5432 \
            -v netbox-set-postgres-data:/var/lib/postgresql/data:Z \
            -e POSTGRES_USER=netbox \
            -e POSTGRES_PASSWORD=your_secure_password \
            -e POSTGRES_DB=netboxdb \
            --restart=unless-stopped \
            postgres:17-alpine
        ```

4. Check the PostgreSQL container is running on designed ports and IP:

    ```bash
    podman ps -a
    ```

5. Check the PostgreSQL logs:

    ```bash
    podman logs netbox-set-postgres-service
    ```

6. Check the PostgreSQL database:

    ```bash
    podman exec -it netbox-set-postgres-service psql -U netbox -d netboxdb
    ```

    - In postgres prompt:

    ```bash
    netboxdb=# \l
    ```

##### Redis

Redis is the cache and queue tasks backend for **NetBox**. The official doc says:

  *Note that NetBox requires the specification of two separate Redis databases: tasks and caching. These may both be provided by the same Redis service, however each should have a unique numeric database ID.*

and

  *It is highly recommended to keep the task and cache databases separate. Using the same database number on the same Redis instance for both may result in queued background tasks being lost during cache flushing events.*

1. Use `redis` image to install **Redis**:

    ```bash
    podman pull docker.io/valkey/valkey:8.0-alpine
    ```

2. Create the volumes for the **Redis** task queue and cache:

    ```bash
    podman volume create netbox-set-redis-task-queue-data
    podman volume create netbox-set-redis-cache-data
    ```

3. Set secrets for **Environment variables** with podman:

    ```bash
    printf "your_secure_password" | podman secret create netbox_set_redis_task_password -
    printf "your_secure_password" | podman secret create netbox_set_redis_cache_password -
    ```

4. Deploy the Redis **task queue** container:

    - With secrets and password:

        ```bash
        podman run --name netbox-set-redis-tasks-service \
            -d \
            --network netbox-service-set \
            --ip 10.89.1.32 \
            -p 6379:6379 \
            -v netbox-set-redis-task-queue-data:/data:Z \
            --secret netbox_set_redis_task_password,type=env,target=REDIS_PASSWORD \
            --restart=unless-stopped \
            valkey:8.0-alpine \
            sh -c "valkey-server --appendonly yes --requirepass $REDIS_PASSWORD"
        ```

    - Optionally test without secrets and password:

        ```bash
        podman run --name netbox-set-redis-tasks-service \
            -d \
            --network netbox-service-set \
            --ip 10.89.1.32 \
            -p 6379:6379 \
            -v netbox-set-redis-task-queue-data:/data:Z \
            --restart=unless-stopped \
            valkey:8.0-alpine \
            sh -c "valkey-server --appendonly yes"
        ```

5. Check the Redis **task queue** container is running on designed ports and IP:

    ```bash
    podman ps -a
    ```

6. Check the Redis **task queue** logs:

    ```bash
    podman logs netbox-set-redis-tasks-service
    ```

7. Check with valkey-cli:

    ```bash
    podman exec -it netbox-set-redis-tasks-service valkey-cli --pass "your_secure_password" ping
    ```

    You need to receive a `PONG` response.

8. Deploy the Redis **cache** container:

    - With secrets and password:

        ```bash
        podman run --name netbox-set-redis-cache-service \
            -d \
            --network netbox-service-set \
            --ip 10.89.1.33 \
            -p 6380:6379 \
            -v netbox-set-redis-task-queue-data:/data:Z \
            --secret netbox_set_redis_task_password,type=env,target=REDIS_PASSWORD \
            --restart=unless-stopped \
            valkey:8.0-alpine \
            sh -c "valkey-server --requirepass $REDIS_PASSWORD"
        ```

    - Optionally test without secrets and password:

        ```bash
        podman run --name netbox-set-redis-cache-service \
            -d \
            --network netbox-service-set \
            --ip 10.89.1.33 \
            -p 6380:6379 \
            -v netbox-set-redis-task-queue-data:/data:Z \
            --restart=unless-stopped \
            valkey:8.0-alpine \
            sh -c "valkey-server"
        ```

9. Check the same as before.

##### Netbox

1. Use `netbox` image to install **Netbox**:

    ```bash
    podman pull netboxcommunity/netbox:latest-3.2.0
    ```

2. Create the persistent volumes for media, reports ans scripts:

    ```bash
    podman volume create netbox-set-media-data
    podman volume create netbox-set-reports-data
    podman volume create netbox-set-scripts-data
    ```

3. Set secrets for **Environment variables** with podman:

    ```yml
    # ALLOWED_HOSTS section
    ALLOWED_HOSTS: "10.89.1.30"
    # DATABASE section
    DB_NAME: "netboxdb"
    DB_USER: "netbox"
    DB_PASSWORD: "your_secure_password"
    DB_HOST: "10.89.1.31"
    DB_PORT: "5433" # This is the port of the PostgreSQL container for Netbox in the host machine.
    #DB_WAIT_DEBUG: 1
    # REDIS section
    ## REDIS tasks
    REDIS_HOST: "10.89.1.32"
    REDIS_PORT: "6379"
    #REDIS_PASSWORD: "your_secure_password"
    REDIS_DB: 0
    REDIS_SSL: False
    ## REDIS cache
    REDIS_CACHE_HOST: "10.89.1.33"
    REDIS_CACHE_PORT: "6380"
    #REDIS_CACHE_PASSWORD: "your_secure_password"
    #REDIS_CACHE_DB: 1
    REDIS_CACHE_SSL: False
    # SECRET_KEY section
    SECRET_KEY: "generate_a_secure_key"
    ```

    ```yml
    ALLOWED_HOSTS: "*"
    DB_WAIT_DEBUG: 1
    DB_HOST: "10.89.1.31"
    DB_USER: "netbox"
    DB_PASSWORD: "db_password"
    SECRET_KEY: "secret_key" >= 50 characters
    POSTGRES_DB: netbox-postgres
    POSTGRES_USER: netbox
    POSTGRES_PASSWORD: "db_password"
    REDIS_HOST: "10.89.1.32"
    REDIS_PORT: "6379"
    SUPERUSER_NAME: "superuser_name"
    SUPERUSER_EMAIL: "superuser_email"
    SUPERUSER_PASSWORD: "superuser_password"
    SKIP_SUPERUSER: "false"
    ```

4. Create the secrets for **Environment variables** with podman:

    ```bash


5. Deploy the Netbox container:

    - With secrets:

        ```bash
        podman run --name netbox-set-netbox-service \
            -d \
            --network netbox-service-set \
            --ip

<!--
    - Without secrets:

        ```bash
        podman run --name netbox-set-netbox-service \
            -d \
            --network netbox-service-set \
            --ip 10.89.1.30 \
            -p 8000:8000 \
            -v netbox-set-media-data:/opt/netbox/netbox/media:Z \
            -v netbox-set-reports-data:/opt/netbox/netbox/reports:Z \
            -v netbox-set-scripts-data:/opt/netbox/netbox/scripts:Z \
            -e ALLOWED_HOSTS="10.89.1.30" \
            -e DB_NAME="netboxdb" \
            -e DB_USER="netbox" \
            --secret netbox_set_postgres_password,type=env,target=DB_PASSWORD \
            -e DB_HOST="10.89.1.31" \
            -e DB_PORT="5433" \
            -e DB_WAIT_DEBUG=1 \
            -e REDIS_HOST="10.89.1.32" \
            -e REDIS_PORT="6379" \
            #--secret netbox_set_redis_task_password,type=env,target=REDIS_PASSWORD \
            -e REDIS_DB=0 \
            -e REDIS_SSL=False \
            -e REDIS_CACHE_HOST="10.89.1.33" \
            -e REDIS_CACHE_PORT="6380" \
            #--secret netbox_set_redis_cache_password,type=env,target=REDIS_CACHE_PASSWORD \
            -e REDIS_CACHE_DB=1 \
            -e REDIS_CACHE_SSL=False \
            -e SECRET_KEY="^Juy^bAT2bmFRYVnJHVg0&YkkFyM=-PODj*4zZM@th2@C)_$Jv:" \
            --restart=unless-stopped \
            netbox:latest-3.2.0
        ```
-->

    - With secrets and with two Redis without password:

        ```bash
        podman run --name netbox-set-netbox-service \
            -d \
            --network netbox-service-set \
            --ip 10.89.1.30 \
            -p 8000:8000 \
            -v netbox-set-media-data:/opt/netbox/netbox/media:Z \
            -v netbox-set-reports-data:/opt/netbox/netbox/reports:Z \
            -v netbox-set-scripts-data:/opt/netbox/netbox/scripts:Z \
            -e ALLOWED_HOSTS="10.89.1.30" \
            -e DB_NAME="netboxdb" \
            -e DB_USER="netbox" \
            --secret netbox_set_postgres_password,type=env,target=DB_PASSWORD \
            -e DB_HOST="10.89.1.31" \
            -e DB_PORT="5433" \
            -e DB_WAIT_DEBUG=1 \
            -e REDIS_HOST="10.89.1.32" \
            -e REDIS_PORT="6379" \
            -e REDIS_DB=0 \
            -e REDIS_SSL=False \
            -e REDIS_CACHE_HOST="10.89.1.33" \
            -e REDIS_CACHE_PORT="6380" \
            -e REDIS_CACHE_DB=1 \
            -e REDIS_CACHE_SSL=False \
            -e SECRET_KEY="^Juy^bAT2bmFRYVnJHVg0&YkkFyM=-PODj*4zZM@th2@C)_$Jv:" \
            --restart=unless-stopped \
            netbox:latest-3.2.0
        ```

    - With secrets and with single Redis without password:

        ```bash
        podman run --name netbox-set-netbox-service \
            -d \
            --network netbox-service-set \
            --ip 10.89.1.30 \
            -p 8000:8080 \
            -v netbox-set-media-data:/opt/netbox/netbox/media:Z \
            -v netbox-set-reports-data:/opt/netbox/netbox/reports:Z \
            -v netbox-set-scripts-data:/opt/netbox/netbox/scripts:Z \
            -e ALLOWED_HOSTS="*" \
            -e DB_WAIT_DEBUG=1 \
            -e DB_HOST="10.89.1.31" \
            -e DB_USER="netbox" \
            -e DB_NAME="netboxdb" \
            --secret netbox_set_postgres_password,type=env,target=DB_PASSWORD \
            -e SECRET_KEY="secretkey" \
            -e DB_PORT="5432" \
            -e REDIS_HOST="10.89.1.32" \
            -e REDIS_PORT="6379" \
            -e SUPERUSER_NAME="admin" \
            -e SUPERUSER_EMAIL="jorge.diazsagot@ucr.ac.cr" \
            -e SUPERUSER_PASSWORD="mgsg2014" \
            -e SKIP_SUPERUSER="false" \
            --restart=unless-stopped \
            netbox:latest-3.2.0
        ```

1. Check the Netbox container is running on designed ports and IP:

    ```bash
    podman ps -a
    ```

2. Check the Netbox logs:

    ```bash
    podman logs netbox-set-netbox-service
    ```

### vm-reproxy

**Purpose:**
This VM is used to run as a reverse proxy for the web application services WikiJS and Netbox. It is used to run the following tools:

- Nginx (reverse proxy) container.
- Podman (for running the web applications)
- Cockpit inheritance from base image with podman and file manager addons.

#### Oracle VirtualBox Network settings

Adapter 1: **NAT** (for internet access) with port forwarding for WikiJS (8080:80) and Netbox (8000:80).

- This is for the `ExtNetHome` network in the diagram.

Adapter 2: **Internal Network** adapter for communication with the other VMs.

- This is for the `IntNetMgmt` network in the diagram, to manage the other VMs.

Adapter 3: **Internal Network** adapter for communication with the web applications.

- This is for the `IntNetSrv` network in the diagram, to communication with service VMs.


### Cockpit web console

We can access to all the VMs with the web console.

1. We need to access to the web console of the VM `vm-mgmt` with the IP address of the VM and port 9090.
2. Then we can access to the other VMs with the IP address of the VM and port 9090 as `admin` user.

    See the image below:
    ![alt text](../media/Installation-Guide/cockpit-add-host.png)

3. 

## TIPS

### Oracle VirtualBox

#### See vms IP addresses

- VirtualBox in Windows Host: [Reference](https://linux.how2shout.com/how-to-find-the-ip-address-of-a-guest-in-virtualbox/#On_Windows_Guest_OS)

For ipv4 address:

```bash
PS C:\Program Files\Oracle\VirtualBox> ./VBoxManage guestproperty get "guestvm" /VirtualBox/GuestInfo/Net/0/V4/IP
```

For ipv6 address:

```bash
PS C:\Program Files\Oracle\VirtualBox> ./VBoxManage guestproperty get "guestvm" /VirtualBox/GuestInfo/Net/0/V6/IP
```
