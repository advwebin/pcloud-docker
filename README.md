# pcloudcc_docker

> This is a fork of [jlloyola/pcloudcc_docker](https://github.com/jlloyola/pcloudcc_docker). All credit for the original implementation goes to the original author.
<p>
<a href="https://github.com/jlloyola/pcloudcc_docker/actions"><img alt="Actions Status" src="https://github.com/jlloyola/pcloudcc_docker/actions/workflows/docker-image.yml/badge.svg"></a>
<a href="https://hub.docker.com/r/jloyola/pcloudcc"><img alt="Docker pulls" src="https://img.shields.io/docker/pulls/jloyola/pcloudcc"></a>
<a href="https://github.com/jlloyola/pcloudcc_docker/blob/main/LICENSE"><img alt="License: GPL-3.0" src="https://img.shields.io/github/license/jlloyola/pcloudcc_docker"></a>
</p>

This repo defines a Dockerfile to build the
[pcloud console client](https://github.com/pcloudcom/console-client)
from source and run it inside a container.  

The container exposes your pcloud drive in a location of your
choice. It runs using non-root `uid`:`gid` to properly handle
file permissions and allow seamless file transfers between the host, the container, and pcloud.  
This image includes PR [#163](https://github.com/pcloudcom/console-client/pull/163)
to enable one-time password multi-factor authentication.

## Setup instructions
It is recommended to use a compose file to simplify setup.  
Make sure you have docker installed. Refer to https://docs.docker.com/engine/install/

### 1. Obtain your user and group ID
You can get them from the command line. For example
```
% id -u
1000
% id -g
1000
```
### 2. Create a `.env` file
Enter the relevant information for your setup:
| Variable           | Purpose                                    | Sample Value          |
|----------------    |--------------------------------------------|-----------------------|
|PCLOUD_IMAGE        |Image version                               |jloyola/pcloudcc:latest|
|PCLOUD_DRIVE        |Host directory where pcloud will be mounted |/home/user/pCloudDrive |
|PCLOUD_USERNAME     |Your pcloud username                        |example@example.com    |
|PCLOUD_UID          |Your host user id (obtained above)          |1000                   |
|PCLOUD_GID          |Your host group id (obtained above)         |1000                   |
|PCLOUD_SAVE_PASSWORD|Save password in cache volume               |1                      |

Example `.env` file:  
```
PCLOUD_IMAGE=jloyola/pcloudcc:latest
PCLOUD_DRIVE=/home/user/pCloudDrive
PCLOUD_USERNAME=example@example.com
PCLOUD_UID=1000
PCLOUD_GID=1000
PCLOUD_SAVE_PASSWORD=1
```
### 3. Create the pcloud directory on the host
```
mkdir -p <PCLOUD_DRIVE>
```
`<PCLOUD_DRIVE>` corresponds to the same directory you specified in the `.env`
This step guarantees the directory permissions for the pcloud mount point
match your `uid:gid`. Otherwise, Docker will create the folder, and the
folder will be given 'root' permissions, which then causes the Docker container to fail upon startup with the following error message:
```
ROOT level privileges prohibited!
```
### 4. Initial run
Copy the [compose.yml](https://github.com/jlloyola/pcloudcc_docker/blob/main/compose.yml)
file from this repo and place it in the same location as
your `.env` file
You will need to login the initial run.
```
% docker compose up -d
[+] Running 1/0
 ⠿ Container pcloudcc_docker-pcloud-1
```
Attach to the container to enter your password (and MFA code if enabled).  
Hit enter after attaching to the container to be prompted for your credentials
```
% docker attach pcloudcc_docker-pcloud-1
Down: Everything Downloaded| Up: Everything Uploaded, status is LOGIN_REQUIRED
Please, enter password
logging in
Down: Everything Downloaded| Up: Everything Uploaded, status is CONNECTING
Down: Everything Downloaded| Up: Everything Uploaded, status is TFA_REQUIRED
Please enter 2fa code
123456
Down: Everything Downloaded| Up: Everything Uploaded, status is CONNECTING
Down: Everything Downloaded| Up: Everything Uploaded, status is SCANNING
Down: Everything Downloaded| Up: Everything Uploaded, status is READY
```
### 5. Access your pcloud drive
You can now access your pcloud drive from your host machine
at the location specified in your `.env` file.

## Troubleshooting
When stopping the container, the mount point can get stuck.  
Run the following command to fix it
```
% fusermount -uz <PCLOUD_DRIVE>
```
`<PCLOUD_DRIVE>` corresponds to the same directory you specified in the `.env`.  
See https://stackoverflow.com/a/25986155
## Acknowledgments
The code in this repo was inspired by the work from:
* zcalusic: https://github.com/zcalusic/dockerfiles/tree/master/pcloud
* abraunegg: https://github.com/abraunegg/onedrive

---

## Setup on ZimaOS

This guide explains how to run pcloud on ZimaOS using the Custom App feature with proper handling of environment variables and sensitive data.

### Understanding ZimaOS Paths

ZimaOS uses specific paths for Docker volumes:
- **ZimaOS storage path**: `/mnt/zfspool/`
- **Container internal path**: `/pCloudDrive` (fixed inside container)

### Environment Variables Reference

| Variable | Required | Sensitive | Description | Default |
|----------|----------|-----------|-------------|---------|
| `PCLOUD_USERNAME` | No* | No | Your pcloud email address | - |
| `PCLOUD_UID` | Yes | No | Your host user ID (see below) | - |
| `PCLOUD_GID` | Yes | No | Your host group ID (see below) | - |
| `PCLOUD_SAVE_PASSWORD` | No | No | Set to `1` to cache credentials | - |
| `PCLOUD_PASSWORD_FILE` | No | Yes | Path to password file (see below) | - |
| `PCLOUD_DEBUG` | No | No | Set to `1` for debug output | `0` |

*Required for automatic login. If not provided, you'll be prompted interactively.

### Security Options for Password

There are **three methods** to provide your pcloud password, ordered by security level:

#### Method 1: Interactive Input (Most Secure)

Let the container prompt for password on first run. No password stored anywhere.

```yaml
services:
  pcloud:
    image: jloyola/pcloudcc:latest
    container_name: pcloud
    restart: unless-stopped
    stdin_open: true
    tty: true
    volumes:
      - type: volume
        source: pcloud_cache
        target: /home/pcloud/.pcloud
      - type: bind
        source: /mnt/zfspool/pCloudDrive
        target: /pCloudDrive
        bind:
          propagation: shared
    devices:
      - /dev/fuse:/dev/fuse
    cap_add:
      - SYS_ADMIN
    environment:
      - PCLOUD_UID=1000
      - PCLOUD_GID=1000

volumes:
  pcloud_cache:
```

#### Method 2: Password File (Recommended for Automation)

Create a file containing your password and mount it into the container. The file should contain only your password (no extra lines).

**Step 2.1: Create password file on ZimaOS**

```bash
# Create a directory for secrets
mkdir -p /mnt/zfspool/pcloud-secrets

# Create the password file (replace with your actual password)
echo -n "your_pcloud_password" > /mnt/zfspool/pcloud-secrets/pcloud_pass.txt
```

**Step 2.2: Use in Docker Compose**

```yaml
services:
  pcloud:
    image: jloyola/pcloudcc:latest
    container_name: pcloud
    restart: unless-stopped
    stdin_open: true
    tty: true
    volumes:
      - type: volume
        source: pcloud_cache
        target: /home/pcloud/.pcloud
      - type: bind
        source: /mnt/zfspool/pCloudDrive
        target: /pCloudDrive
        bind:
          propagation: shared
      - type: bind
        source: /mnt/zfspool/pcloud-secrets
        target: /secrets
        read_only: true
    devices:
      - /dev/fuse:/dev/fuse
    cap_add:
      - SYS_ADMIN
    environment:
      - PCLOUD_USERNAME=your_email@example.com
      - PCLOUD_UID=1000
      - PCLOUD_GID=1000
      - PCLOUD_SAVE_PASSWORD=1
      - PCLOUD_PASSWORD_FILE=/secrets/pcloud_pass.txt

volumes:
  pcloud_cache:
```

#### Method 3: Environment Variable (Least Secure - Not Recommended)

**⚠️ WARNING**: This method stores your password in plain text in the compose file. Only use for testing.

```yaml
services:
  pcloud:
    image: jloyola/pcloudcc:latest
    container_name: pcloud
    restart: unless-stopped
    stdin_open: true
    tty: true
    volumes:
      - type: volume
        source: pcloud_cache
        target: /home/pcloud/.pcloud
      - type: bind
        source: /mnt/zfspool/pCloudDrive
        target: /pCloudDrive
        bind:
          propagation: shared
    devices:
      - /dev/fuse:/dev/fuse
    cap_add:
      - SYS_ADMIN
    environment:
      - PCLOUD_USERNAME=your_email@example.com
      - PCLOUD_PASSWORD=your_password_here
      - PCLOUD_UID=1000
      - PCLOUD_GID=1000
      - PCLOUD_SAVE_PASSWORD=1
    # ⚠️ WARNING: Password visible in docker-compose.yml!
```

### Complete Setup Guide for ZimaOS

#### Step 1: Find Your UID and GID

1. Open **Settings** → **Users** in ZimaOS
2. Click on your user account
3. Note the UID (User ID) and GID (Group ID)
4. Default is typically `1000:1000` for the first user

Or via terminal:
```bash
id
# Output: uid=1000(user) gid=1000(user) groups=1000(user)
```

#### Step 2: Create Storage Directories

1. Go to **Settings** → **Storage** in ZimaOS
2. Create a folder named `pCloudDrive`:
   - Click **+ Add Folder**
   - Name: `pCloudDrive`
   - Location: `/mnt/zfspool/`
3. **Important**: Delete and recreate the folder if it was created by Docker (root ownership)

For password file (Method 2):
4. Create folder `pcloud-secrets`:
   - Click **+ Add Folder**
   - Name: `pcloud-secrets`
   - Location: `/mnt/zfspool/`

#### Step 3: Add Custom App in ZimaOS

1. Navigate to **Apps** in ZimaOS
2. Click the **+** button (top right)
3. Select **Add Custom App**
4. Choose **Docker Compose** tab
5. Paste your compose configuration (see methods above)
6. Click **Install**

#### Step 4: Initial Login (Interactive Method)

If you used Method 1 (interactive):

1. After container starts, go to **Apps**
2. Find **pcloud** in your custom apps list
3. Click the **Terminal** icon (console)
4. You should see:
   ```
   Please, enter password
   ```
5. Type your password and press **Enter**
6. If MFA is enabled, you'll see:
   ```
   Please enter 2fa code
   ```
7. Enter your 2FA code and press **Enter**
8. Wait for status to show: `READY`

**Status Messages Explained**:
| Status | Meaning |
|--------|---------|
| `LOGIN_REQUIRED` | Waiting for password |
| `CONNECTING` | Authenticating with pcloud |
| `TFA_REQUIRED` | Waiting for 2FA code |
| `SCANNING` | Scanning remote file structure |
| `READY` | Connected and syncing |

#### Step 5: Verify Access

Your pcloud drive is now mounted:
- **Inside container**: `/pCloudDrive`
- **On ZimaOS**: `/mnt/zfspool/pCloudDrive`

Access methods:
- ZimaOS File Manager → `/mnt/zfspool/pCloudDrive`
- Network share (Samba/NFS) via ZimaOS Settings
- Other apps on ZimaOS can access `/mnt/zfspool/pCloudDrive`

---

### ZimaOS-Specific Configuration

#### Using Docker Secrets (Advanced)

For maximum security, use Docker secrets with ZimaOS:

```yaml
services:
  pcloud:
    image: jloyola/pcloudcc:latest
    container_name: pcloud
    restart: unless-stopped
    stdin_open: true
    tty: true
    volumes:
      - type: volume
        source: pcloud_cache
        target: /home/pcloud/.pcloud
      - type: bind
        source: /mnt/zfspool/pCloudDrive
        target: /pCloudDrive
        bind:
          propagation: shared
    devices:
      - /dev/fuse:/dev/fuse
    cap_add:
      - SYS_ADMIN
    environment:
      - PCLOUD_USERNAME=your_email@example.com
      - PCLOUD_UID=1000
      - PCLOUD_GID=1000
      - PCLOUD_SAVE_PASSWORD=1
    secrets:
      - pcloud_password

secrets:
  pcloud_password:
    file: /mnt/zfspool/pcloud-secrets/pcloud_pass.txt

volumes:
  pcloud_cache:
```

#### Enabling Debug Mode

To troubleshoot issues:

```yaml
environment:
  - PCLOUD_DEBUG=1
  - PCLOUD_UID=1000
  - PCLOUD_GID=1000
```

Check logs in ZimaOS Apps section for debug output.

---

### Important ZimaOS Notes

#### FUSE Device Requirement

The container requires `/dev/fuse` for pcloud FUSE mount to work:
- `devices: - /dev/fuse:/dev/fuse` - Passes FUSE device
- `cap_add: - SYS_ADMIN` - Required for FUSE operation

#### File Permission Issues

**⚠️ Critical**: If you see `ROOT level privileges prohibited!`:

1. Delete the `pCloudDrive` folder in ZimaOS Storage
2. Recreate it through ZimaOS UI (not via terminal/Docker)
3. Verify ownership: folder should be owned by your user (1000:1000)

#### Container Restart Behavior

- With `PCLOUD_SAVE_PASSWORD=1`: Credentials cached, automatic reconnection
- Without: Requires interactive login on each restart

#### Cleaning Up Stuck Mounts

If mount point gets stuck after container stop:
```bash
fusermount -uz /mnt/zfspool/pCloudDrive
```

Run this in ZimaOS terminal (Settings → Terminal).

---

### Troubleshooting on ZimaOS

| Issue | Symptom | Solution |
|-------|---------|----------|
| `ROOT level privileges prohibited!` | Container exits immediately | Delete/recreate pCloudDrive folder in ZimaOS UI |
| Container won't start | FUSE error in logs | Enable FUSE in BIOS/UEFI, check device mapping |
| Login fails | `AUTH_FAILED` in logs | Check username/password, 2FA code |
| Files not syncing | `CONNECTED` but empty | Check container logs, network connectivity |
| Mount stuck | Folder not accessible after stop | Run `fusermount -uz /mnt/zfspool/pCloudDrive` |
| Permission denied | Can't write to folder | Verify UID/GID matches your ZimaOS user |

#### Checking Container Logs

1. Go to **Apps** in ZimaOS
2. Find **pcloud** custom app
3. Click **Logs** icon

Or via terminal:
```bash
docker logs pcloud
```

---

### Building for ZimaOS Devices

ZimaOS runs on various hardware. Build accordingly:

| Device | Architecture | Build Command |
|--------|--------------|--------------|
| ZimaBoard (original) | ARM64 | `linux/arm64` |
| ZimaBoard 2 | ARM64 | `linux/arm64` |
| ZimaCube | x86-64 | `linux/amd64` |
| ZimaCube Pro | x86-64 | `linux/amd64` |

```bash
# Build for ARM64 (ZimaBoard, ZimaCube)
docker buildx build --platform linux/arm64 -t my-pcloud:latest .

# Build for x86-64 (ZimaCube)
docker buildx build --platform linux/amd64 -t my-pcloud:latest .

# Build for all supported platforms
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 -t my-pcloud:latest .
```

Push to your registry:
```bash
docker tag my-pcloud:latest your-registry/pcloud:latest
docker push your-registry/pcloud:latest
```

Then use your custom image in the compose file.
