# SFTPGo Setup Guide for Windows (Dual Instance Topology)

This document provides step-by-step instructions for running two independent SFTPGo instances on Windows to serve as **Server A (Source SFTP)** and **Server B (Destination SFTP)** for the MFT Integration Engine.

---

## 1. Prerequisites & Download

SFTPGo is an open-source, full-featured SFTP server written in Go.

1. Download the latest Windows release archive (e.g. `sftpgo_<version>_windows_x86_64.zip`) from the official GitHub releases:
   https://github.com/drakkan/sftpgo/releases
2. Extract the archive into a dedicated directory, for example:
   `C:\sftpgo` (or `D:\sftpgo`)

---

## 2. Directory Structure for Dual Instances

To run two fully isolated SFTP instances with separate databases, host keys, configuration directories, and user data roots, create the following layout:

```text
C:\sftpgo\
├── sftpgo.exe
├── instance-a/
│   ├── config/
│   │   └── sftpgo.json
│   ├── data/
│   │   └── mft_source/
│   │       └── upload/
│   └── sftpgo.db
└── instance-b/
    ├── config/
    │   └── sftpgo.json
    ├── data/
    │   └── mft_destination/
    │       └── incoming/
    └── sftpgo.db
```

PowerShell setup commands:
```powershell
New-Item -ItemType Directory -Force -Path "C:\sftpgo\instance-a\config", "C:\sftpgo\instance-a\data\mft_source\upload"
New-Item -ItemType Directory -Force -Path "C:\sftpgo\instance-b\config", "C:\sftpgo\instance-b\data\mft_destination\incoming"
```

---

## 3. Configuration Setup

### Instance A (Source SFTP - Port 2022)
Create `C:\sftpgo\instance-a\config\sftpgo.json`:
```json
{
  "common": {
    "idle_timeout": 15
  },
  "sftpd": {
    "bindings": [
      {
        "port": 2022,
        "address": "0.0.0.0"
      }
    ]
  },
  "httpd": {
    "bindings": [
      {
        "port": 9080,
        "address": "0.0.0.0"
      }
    ]
  },
  "data_provider": {
    "driver": "sqlite",
    "name": "C:\\sftpgo\\instance-a\\sftpgo.db"
  }
}
```

### Instance B (Destination SFTP - Port 2222)
Create `C:\sftpgo\instance-b\config\sftpgo.json`:
```json
{
  "common": {
    "idle_timeout": 15
  },
  "sftpd": {
    "bindings": [
      {
        "port": 2222,
        "address": "0.0.0.0"
      }
    ]
  },
  "httpd": {
    "bindings": [
      {
        "port": 9081,
        "address": "0.0.0.0"
      }
    ]
  },
  "data_provider": {
    "driver": "sqlite",
    "name": "C:\\sftpgo\\instance-b\\sftpgo.db"
  }
}
```

*Note: HTTP admin ports are mapped to `9080` and `9081` to prevent port collisions with Spring Boot (`8080`).*

---

## 4. Starting the Instances

Open two independent Windows PowerShell terminal windows:

### Terminal 1: Start Instance A
```powershell
cd C:\sftpgo
.\sftpgo.exe serve --config-dir "C:\sftpgo\instance-a\config"
```

### Terminal 2: Start Instance B
```powershell
cd C:\sftpgo
.\sftpgo.exe serve --config-dir "C:\sftpgo\instance-b\config"
```

---

## 5. Configuring Users via WebAdmin

### Concept: SFTPGo Admin vs. SFTP User
- **Admin User:** Web administrator account managing server settings, folders, and users. Does not transfer files over SFTP.
- **SFTP User:** Service account authenticating over SSH/SFTP (port 2022 / 2222) restricted to their virtual home directory.

### Step 5.1: Instance A (Source)
1. Open your browser: `http://localhost:9080/web/admin`
2. First-time setup: Create an admin user (e.g. `admin` / `AdminSecretPass123!`).
3. In the sidebar, navigate to **Users** -> **Add User (+)**.
4. Enter:
   - **Username:** `mft_source`
   - **Password:** `source_pass` (or your preferred secure password)
   - **Home directory:** `C:\sftpgo\instance-a\data\mft_source`
   - **Permissions:** Check `list`, `download`, `quota-scans`.
5. Click **Submit**.

### Step 5.2: Instance B (Destination)
1. Open your browser: `http://localhost:9081/web/admin`
2. First-time setup: Create an admin user.
3. In the sidebar, navigate to **Users** -> **Add User (+)**.
4. Enter:
   - **Username:** `mft_destination`
   - **Password:** `dest_pass`
   - **Home directory:** `C:\sftpgo\instance-b\data\mft_destination`
   - **Permissions:** Check `list`, `upload`, `overwrite`, `rename`, `create_dirs`.
5. Click **Submit**.

---

## 6. Setting Environment Variables in Windows PowerShell

Prior to launching the Spring Boot application or tests, set the passwords in PowerShell:

```powershell
$env:SFTP_SOURCE_HOST = "localhost"
$env:SFTP_SOURCE_PORT = "2022"
$env:SFTP_SOURCE_USERNAME = "mft_source"
$env:SFTP_SOURCE_PASSWORD = "source_pass"

$env:SFTP_DESTINATION_HOST = "localhost"
$env:SFTP_DESTINATION_PORT = "2222"
$env:SFTP_DESTINATION_USERNAME = "mft_destination"
$env:SFTP_DESTINATION_PASSWORD = "dest_pass"
```

---

## 7. Verifying SFTP Connectivity

Verify using PowerShell or OpenSSH client:

```powershell
# Test Source Instance
sftp -P 2022 mft_source@localhost

# Test Destination Instance
sftp -P 2222 mft_destination@localhost
```

Or test port binding using PowerShell:
```powershell
Test-NetConnection -ComputerName localhost -Port 2022
Test-NetConnection -ComputerName localhost -Port 2222
```
