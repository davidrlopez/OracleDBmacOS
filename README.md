# Oracle Database 23c Free on macOS (Apple Silicon & Intel)

A complete guide to running Oracle Database 23c Free locally on macOS using Colima and Docker, with full support for Apple Silicon (M1/M2/M3/M4) via Rosetta 2 emulation.

## The Problem

Oracle does not provide a native ARM64 Docker image for their databases. Running Oracle DB on modern Macs via standard Docker Desktop often leads to architecture mismatch errors (`linux/amd64` vs `linux/arm64`) or significant performance degradation.

## The Solution

This guide uses [Colima](https://github.com/abiosoft/colima), a lightweight container runtime for macOS. By configuring Colima to use Apple's Virtualization Framework with Rosetta 2, we emulate the `x86_64` architecture required by the Oracle image with near-native performance — no Docker Desktop required.

---

## Prerequisites

- macOS Sequoia or later (Apple Silicon or Intel)
- At least 8 GB RAM recommended (3.5 GB minimum allocated to the VM)
- [Homebrew](https://brew.sh) — all dependencies are installed through it
- [Oracle Container Registry](https://container-registry.oracle.com) account — required to pull the official image
- A SQL client: [SQL Developer](https://www.oracle.com/database/sqldeveloper/), [DBeaver](https://dbeaver.io), or `sqlcl`

If you do not have Homebrew installed:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

---

## Installation

### 1. Install Colima and Docker CLI

```bash
brew install colima docker
```

### 2. Start Colima with x86_64 Emulation

Oracle Database 23c Free is only available as a Linux x86_64 image. Colima must be started with Rosetta-based emulation to run it on Apple Silicon.

```bash
colima start \
  --arch x86_64 \
  --vm-type vz \
  --vz-rosetta \
  --mount-type virtiofs \
  --memory 3
```

| Flag | Description |
|---|---|
| `--arch x86_64` | Emulates x86_64 architecture via Rosetta 2 |
| `--vm-type vz` | Uses macOS Virtualization Framework |
| `--vz-rosetta` | Enables Rosetta 2 translation layer |
| `--mount-type virtiofs` | High-performance file sharing between host and VM |
| `--memory 3` | Allocates 3 GB RAM to the VM (minimum required) |

Verify Colima is running correctly:

```bash
colima status
```

Expected output should confirm `arch: x86_64` and `runtime: docker`.

### 3. Pull Oracle Database 23c Free

Log in to the Oracle Container Registry. An Oracle account is required:

```bash
docker login container-registry.oracle.com
```

Pull the official image (~9.5 GB):

```bash
docker pull container-registry.oracle.com/database/free
```

### 4. Run the Container

```bash
docker run -d \
  --name ora23c \
  -p 1521:1521 \
  --mount source=oradata,target=/opt/oracle/oradata \
  container-registry.oracle.com/database/free
```

> If port 1521 is already in use on your machine, change the mapping to `-p 1522:1521` and use port `1522` in your client.

Monitor the logs and wait for the database to be ready:

```bash
docker logs -f ora23c
```

Wait for the message: `DATABASE IS READY TO USE!`

### 5. Set the Database Password

Replace `YourStrongPassword123` with your desired password:

```bash
docker exec -it ora23c ./setPassword.sh YourStrongPassword123
```

---

## Connecting to the Database

Use any Oracle-compatible client (SQL Developer, DBeaver, sqlcl) with the following parameters:

| Field | Value |
|---|---|
| Hostname | `localhost` |
| Port | `1521` (or `1522` if you changed the mapping) |
| Service Name / SID | `FREE` |
| Username | `SYS` |
| Role | `SYSDBA` |
| Password | The password set in step 5 |

### Creating a Working Schema

When connected as `SYS`, alter the session before creating users:

```sql
ALTER SESSION SET "_ORACLE_SCRIPT"=true;
```

Create a new user (identifiers must be in **UPPERCASE** to avoid connection errors):

```sql
CREATE USER MY_USER IDENTIFIED BY "MyPassword123"
  DEFAULT TABLESPACE "USERS"
  TEMPORARY TABLESPACE "TEMP"
  QUOTA 20M ON "USERS";

GRANT ALL PRIVILEGES TO MY_USER;
```

Reconnect using the new user credentials to work within your schema.

---

## Daily Usage

### Start

```bash
colima start \
  --arch x86_64 \
  --vm-type vz \
  --vz-rosetta \
  --mount-type virtiofs \
  --memory 3

docker start ora23c
```

### Stop

```bash
colima stop
```

> Stopping Colima also stops all running containers. Database data is persisted in the `oradata` Docker volume and will be available on the next start.

---

## Troubleshooting

**Connection refused on port 1521 / 1522**
The database may still be initializing. Run `docker logs -f ora23c` and wait for `DATABASE IS READY TO USE!`.

**ORA-01017: invalid credential**
Ensure the username is in uppercase. Oracle identifiers are case-sensitive when quoted.

**Architecture mismatch error on `docker pull`**
Colima may not be running with the correct architecture. Verify with `colima status` that `arch: x86_64` is set.

**Colima fails to start**
Ensure Rosetta 2 is installed:

```bash
softwareupdate --install-rosetta
```

**Container exits immediately**
The VM may not have enough memory. Try increasing `--memory` to `4` or higher.

---

## References

- [Colima — GitHub](https://github.com/abiosoft/colima)
- [Oracle Database Free — Container Registry](https://container-registry.oracle.com/ords/ocr/ba/database/free)
- [Oracle SQL Developer — Download](https://www.oracle.com/database/sqldeveloper/)
- [DBeaver — Download](https://dbeaver.io)
- [Homebrew](https://brew.sh)
- [Rosetta 2 — Apple Support](https://support.apple.com/en-us/102527)
