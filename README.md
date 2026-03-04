# Oracle Database 23c Free on macOS (Apple Silicon & Intel)

A comprehensive guide to running Oracle Database 23c Free natively on macOS using **Colima** and **Docker**, fully supporting Apple Silicon (M1/M2/M3/M4) via Rosetta 2 emulation.

## The Problem
Oracle does not provide a native ARM64 Docker image for their databases. Running Oracle DB on modern Macs via standard Docker Desktop often leads to architecture mismatch errors (`linux/amd64` vs `linux/arm64`) or heavy performance degradation. 

## The Solution
We bypass this limitation by using **Colima**, a lightweight container runtime for macOS. By configuring Colima to use Apple's Virtualization Framework and Rosetta 2, we can seamlessly emulate the `x86_64` architecture required by the Oracle image with near-native performance.

---

## Prerequisites

You only need **Homebrew** installed. If you don't have it, open your terminal and run:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Step-by-Step Installation

### 1. Install and Configure Colima

First, install Colima via Homebrew:
```bash
brew install colima
```

Next, start Colima with the specific parameters needed for Oracle DB. We specify the `x86_64` architecture, enable Rosetta 2 emulation, and allocate the minimum required RAM (3GB).
```bash
colima start --arch x86_64 --vm-type=vz --vz-rosetta --mount-type=virtiofs --memory 3
```
*Note: To verify Colima is running correctly, you can use the command `colima status`.*

### 2. Pull and Run Oracle Database 23c

Pull the official Oracle Database 23c Free image:
```bash
docker pull container-registry.oracle.com/database/free
```

Run the container. We map port `1521` for database connections and set up a volume to persist data locally.
```bash
docker run -d \
  --name ora23c \
  -p 1521:1521 \
  --mount source=oradata,target=/opt/oracle/oradata \
  container-registry.oracle.com/database/free
```
*Note: If port 1521 is already in use on your machine, you can change the mapping to `-p 1522:1521`.*

### 3. Set the Database Password

Wait a few moments for the database to initialize (you can check logs with `docker logs -f ora23c` looking for "Database ready to use").

Once ready, access the container to set your sysadmin password (replace `YourStrongPassword123` with your desired password):
```bash
docker exec -it ora23c ./setPassword.sh YourStrongPassword123
```

---

## Connecting to the Database

You can now connect to your locally hosted Oracle database using your preferred client (e.g., **SQL Developer**, **DBeaver**, or **sqlcl**).

**Connection Details:**
- **Hostname:** `localhost`
- **Port:** `1521` (or `1522` if you changed the mapping)
- **Service Name (SID):** `FREE`
- **Username:** `SYS`
- **Role:** `SYSDBA`
- **Password:** The password you set in Step 3.

### Altering the Session (Required for new users)
When connected as `SYS`, before creating new user schemas, you must alter the session:
```sql
ALTER SESSION SET "_ORACLE_SCRIPT"=true;
```

Then you can create a standard user:
```sql
CREATE USER MY_USER IDENTIFIED BY "MyPassword123"
DEFAULT TABLESPACE "USERS"
TEMPORARY TABLESPACE "TEMP"
QUOTA 20M ON "USERS";

GRANT ALL PRIVILEGES TO MY_USER;
```
*Note: Ensure the username is written in **UPPERCASE** to avoid connection errors later.*

---

## Managing the Server

To stop the server and free up resources:
```bash
colima stop
```

To start it again:
```bash
colima start
docker start ora23c
```