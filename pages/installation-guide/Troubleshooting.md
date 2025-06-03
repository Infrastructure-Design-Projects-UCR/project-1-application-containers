
# Troubleshooting

## 1. Errors in my Podman containers

### 1.1. containers acquired locks ERROR

**Description**

Sometimes Podman containers just explode and you get an error related with locks in while refreshing the containers and volumes.

An example of the error is:

```text
[admin@vm-monitor ~]$ podman ps
ERRO[0000] Refreshing container 9a7eacbf7bfa468f0e268f0b7d40270d16f9a2f097c1ac63e695c11363a98e57: acquiring lock 2 for container 9a7eacbf7bfa468f0e268f0b7d40270d16f9a2f097c1ac63e695c11363a98e57: file exists
ERRO[0000] Refreshing container eb4ae8faa7163d881fabf0e51c9a455ba50e368725bd50a3a0045efddb3886a4: acquiring lock 4 for container eb4ae8faa7163d881fabf0e51c9a455ba50e368725bd50a3a0045efddb3886a4: file exists
ERRO[0000] Refreshing volume 9ac72443fe5a65bc6903efdf87e9364b5e0b8ea70c0801bb29e914161920c46e: acquiring lock 0 for volume 9ac72443fe5a65bc6903efdf87e9364b5e0b8ea70c0801bb29e914161920c46e: file exists
ERRO[0000] Refreshing volume monitor-set-grafana-data: acquiring lock 3 for volume monitor-set-grafana-data: file exists
ERRO[0000] Refreshing volume monitor-set-influxdb-data: acquiring lock 1 for volume monitor-set-influxdb-data: file exists
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
```

**Investigation**

1. My first suspect is that when the VM make a screenshot, the Podman containers are not able to be refreshed and they get locked.

   - I test this and I don't get the error, so we can discard this option.

2. I found a GitHub issue related with this error, and it says that is a bug when you use ssh with a user that run podman as rootless user an then exit the session.

    - Reference: https://github.com/containers/podman/issues/16784 

### 1.2. Solutions provided

#### 1.2.1. With systemd

https://github.com/containers/podman/issues/16784#issuecomment-1913686983

**Proven Solution: Running Podman System Service with systemd**

If you want Podman containers to keep running independently of your user session, you can configure the Podman system service to run persistently using systemd.

**Tested with:** `podman version 4.2.0`

**Background**

Podman is a "daemonless" container engine. However, the `podman.service` systemd unit can be used to provide an API endpoint and manage containers in the background. By default, the service waits for API calls for 5 seconds after the session ends, then stops. You can change this behavior so the service runs indefinitely.

**Steps**

1. **Copy the Podman systemd service file:**

    ```bash
    sudo cp /usr/lib/systemd/system/podman.service /etc/systemd/system/
    ```

2. **Edit `/etc/systemd/system/podman.service`** to listen on port 8080 and run indefinitely (`--time=0`):

    ```ini
    [Unit]
    Description=Podman API Service
    Requires=podman.socket
    After=podman.socket
    Documentation=man:podman-system-service(1)
    StartLimitIntervalSec=0

    [Service]
    Type=exec
    KillMode=process
    Environment=LOGGING="--log-level=info"
    ExecStart=/usr/bin/podman $LOGGING system service tcp:0.0.0.0:8080 --time=0
    ```

    > **Note:**  
    > - To restrict the API to localhost, use `tcp:127.0.0.1:8080` instead of `tcp:0.0.0.0:8080`.

3. **Reload systemd and enable the services:**

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable podman.socket podman
    sudo systemctl start podman.socket podman
    ```

Now, the Podman service will remain active and listen for API requests on the specified port until explicitly stopped.

#### 1.2.2. With podman non-rootuser

We take this approach to give a solution to the problem of the containers that are not running when you log out from the SSH session.

https://github.com/containers/podman/issues/16784#issuecomment-1974293441

**Workaround Solution: Enable Lingering for Rootless Podman Users**

If you are running Podman containers as a non-root (rootless) user and want your containers to keep running after you log out, you need to enable "lingering" for that user. By default, rootless Podman containers are tied to the user session and will stop when the user logs out unless lingering is enabled.

**Enable lingering with:**

```bash
sudo loginctl enable-linger <username>
```

Replace `<username>` with the actual username running the containers.

This command creates a file in `/var/lib/systemd/linger/` for the user, allowing their processes (including Podman containers) to continue running after logout, until explicitly stopped or the system is rebooted.

> **Note:** If you are running Podman containers as root, consider using a systemd user service instead.

