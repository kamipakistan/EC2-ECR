# Deploy PostgreSQL on AWS EC2 and Connect via pgAdmin

## Overview

This comprehensive guide provides step-by-step instructions for deploying a PostgreSQL database server on AWS EC2 and connecting to it remotely using pgAdmin. This setup is ideal for development, testing, or small-scale production environments where you need a managed database solution with full control over the database configuration.

### What You'll Learn
- How to provision and configure an AWS EC2 instance
- Installing PostgreSQL 16 on Ubuntu
- Creating databases, users, and tables
- Configuring PostgreSQL for remote connections
- Setting up firewall rules for database access
- Connecting to your database using pgAdmin

### Prerequisites
- An AWS account with appropriate permissions
- pgAdmin installed on your local machine (download from [pgAdmin.org](https://www.pgadmin.org/download/))
- Basic familiarity with the Linux command line
- SSH client (OpenSSH on Linux/Mac, PuTTY or WSL on Windows)

---

# Step 1: Launch EC2 Instance

### 1.1 Open AWS Console

1. Log in to your AWS Management Console at https://console.aws.amazon.com
2. In the top search bar, type "EC2" and select the EC2 service
3. Ensure you're in the correct region (top-right corner) - choose the region closest to your users

### 1.2 Navigate to Launch Instance

1. From the EC2 Dashboard, click **Instances** in the left sidebar
2. Click the orange **Launch Instance** button

### 1.3 Configure Instance Settings

#### Name and Tags
- **Name**: `postgresql-server` (or your preferred name)
- Additional tags can be added later for organization

#### Application and OS Images (AMI)
- **Choose AMI**: Ubuntu Server 22.04 LTS (HVM), SSD Volume Type
- **Architecture**: 64-bit (x86)
- *Why Ubuntu?* It offers excellent package management and widespread community support for PostgreSQL

#### Instance Type
| Instance Type | vCPUs | Memory | Use Case |
|--------------|-------|--------|----------|
| t3.micro | 2 | 1 GiB | Testing, low-traffic development |
| t3.small | 2 | 2 GiB | Light development workloads |
| t3.medium | 2 | 4 GiB | Small production workloads |
| t3.xlarge | 4 | 16 GiB | Moderate production workloads |

**For this guide**: Select `t3.micro` for free tier eligibility or `t3.xlarge` for production-like performance testing.

#### Key Pair (Login)
1. Click **Create new key pair**
2. **Key pair name**: `postgresql-server-key`
3. **Key pair type**: RSA
4. **Private key file format**: `.pem` (for OpenSSH)
5. Click **Create key pair** - this will automatically download the key file

> **IMPORTANT**: Save this key file in a secure location. You cannot download it again. Without it, you cannot access your EC2 instance.

### 1.4 Network Settings

1. Click **Edit** under Network Settings
2. **VPC**: Select your default VPC (or choose a custom one if you have specific networking requirements)
3. **Subnet**: Select any subnet (preferably one that allows public IP assignment)
4. **Auto-assign public IP**: Select **Enable**
5. **Firewall (security groups)**: Select **Create security group**
6. **Security group name**: `postgresql-sg`
7. **Description**: `Security group for PostgreSQL server`

#### Configure Security Group Rules

Add the following inbound rules:

| Type | Protocol | Port Range | Source | Description |
|------|----------|------------|--------|-------------|
| SSH | TCP | 22 | My IP | Secure shell access for administration |
| PostgreSQL | TCP | 5432 | My IP | Database connection from pgAdmin |
| Custom TCP | TCP | 5432 | 0.0.0.0/0 | *Optional* - For public access (use with caution) |

> **Security Note**: Restrict PostgreSQL access (port 5432) to only your IP address whenever possible. If you need to connect from multiple locations, consider using a VPN or setting up a bastion host.

### 1.6 Configure Storage

1. Click **Edit** under Configure Storage
2. **Size**: 
   - `20 GB` for testing/development
   - `50-100 GB` for production workloads
3. **Volume Type**: 
   - `gp3` (general purpose SSD) - good balance of price and performance
   - `io2` for high-performance requirements
4. **Delete on termination**: Enable (unless you need to preserve data after instance termination)

### 1.7 Launch Instance

1. Review all settings carefully
2. Click **Launch Instance**
3. Wait for the instance to enter the **Running** state (usually 1-2 minutes)

### 1.8 Record Important Information

After the instance is running:
1. Go to **Instances** dashboard
2. Click on your `postgresql-server` instance
3. Note down:
   - **Public IPv4 address** (e.g., `54.123.45.67`)
   - **Public IPv4 DNS** (e.g., `ec2-54-123-45-67.compute-1.amazonaws.com`)

---

# Step 2: Connect to EC2 Instance

### 2.1 Set Up SSH Key Permissions

Open a terminal (Linux/Mac) or WSL (Windows) and run:

```bash
# Navigate to your SSH directory
cd ~/.ssh

# If the directory doesn't exist, create it
mkdir -p ~/.ssh
cd ~/.ssh

# Copy the key from Downloads (adjust path if downloaded elsewhere)
cp ~/Downloads/postgresql-server-key.pem .

# Set proper permissions - this is critical for SSH security
chmod 400 postgresql-server-key.pem
```

> **Why chmod 400?** SSH requires private key files to be readable only by the owner. Incorrect permissions will result in a "permissions too open" error.

### 2.2 Establish SSH Connection

```bash
# Connect using the public IP address
ssh -i "postgresql-server-key.pem" ubuntu@<EC2_PUBLIC_IP>

# Example:
ssh -i "postgresql-server-key.pem" ubuntu@54.123.45.67
```

**First-time connection prompt**:
```
The authenticity of host '54.123.45.67 (54.123.45.67)' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```
Type `yes` and press Enter.

### 2.3 Verify Connection

Once connected, you should see a prompt like:
```
ubuntu@ip-xxx-xx-x-xx:~$
```

Run a test command to verify:
```bash
whoami
# Output: ubuntu

pwd
# Output: /home/ubuntu
```

---

# Step 3: Set Hostname (Optional but Recommended)

Setting a custom hostname makes the server easier to identify:

```bash
# Check current hostname
hostname
# Output: ip-xxx-xx-x-xx

# Set new hostname
sudo hostnamectl set-hostname postgresql-server

# Verify the change
hostname
# Output: postgresql-server

# Update /etc/hosts to include the new hostname
echo "127.0.0.1 postgresql-server" | sudo tee -a /etc/hosts

# Log out and back in to see the new hostname in your prompt
exit
ssh -i "postgresql-server-key.pem" ubuntu@<EC2_PUBLIC_IP>
```

Your prompt should now show the custom hostname.

---

# Step 4: Update System Packages

Always update your system before installing new software:

```bash
# Update package index (refresh available packages list)
sudo apt update -y

# Upgrade all installed packages to latest versions
sudo apt full-upgrade -y

# Clean up unnecessary packages
sudo apt autoremove -y

# Check Ubuntu version
lsb_release -a
```

**Expected output**:
```
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.3 LTS
Release:        22.04
Codename:       jammy
```

---

# Step 5: Install PostgreSQL

### 5.1 Install PostgreSQL from Ubuntu Repositories

```bash
# Install PostgreSQL and additional utilities
sudo apt install postgresql postgresql-contrib -y

# Verify PostgreSQL version
psql --version
# Output: psql (PostgreSQL) 14.x (Ubuntu 14.x-x)
```

> **Note**: Ubuntu 22.04 LTS comes with PostgreSQL 14 by default. If you need a newer version (15 or 16), see the next section.

### 5.2 (Optional) Install PostgreSQL 16 from Official Repository

For PostgreSQL 16 (latest stable version as of 2024):

```bash
# Add PostgreSQL official repository
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Update package index
sudo apt update

# Install PostgreSQL 16
sudo apt install postgresql-16 postgresql-contrib-16 -y

# Verify installation
sudo systemctl status postgresql
```

### 5.3 Check PostgreSQL Service Status

```bash
# Check if PostgreSQL is running
sudo systemctl status postgresql

# If not running, start it
sudo systemctl start postgresql

# Enable to start automatically on boot
sudo systemctl enable postgresql
```

**Expected output**:
```
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since ...
```

### 5.4 Switch to PostgreSQL User

PostgreSQL creates a system user named `postgres` for administration:

```bash
# Switch to postgres user
sudo -i -u postgres

# You'll see the prompt change to:
postgres@postgresql-server:~$
```

### 5.5 Access PostgreSQL Command Line

```bash
# Connect to PostgreSQL as postgres user
psql

# You'll see the PostgreSQL prompt:
postgres=#
```

### 5.6 Basic PostgreSQL Commands

Once inside the PostgreSQL shell, you can:

```sql
-- Display version
SELECT version();

-- List all databases
\l

-- List all users/roles
\du

-- Get help
\help

-- Quit PostgreSQL shell
\q
```

---

# Step 6: Create Database and User

### 6.1 Create Database User

Still as the `postgres` user, create a new database user:

```bash
# Connect to PostgreSQL
psql
```

```sql
-- Create a new user with a password
CREATE USER myuser WITH PASSWORD 'StrongPassword123!';

-- Grant login privileges
ALTER USER myuser WITH LOGIN;

-- Check created users
\du
```

**Expected output**:
```
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 myuser    |                                                            | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

### 6.2 Create Database

```sql
-- Create a database owned by myuser
CREATE DATABASE mydb OWNER myuser;

-- List databases to verify
\l
```

Look for `mydb` in the list with `myuser` as the owner.

### 6.3 Grant Privileges

```sql
-- Grant all privileges on the database to the user
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;

-- Grant schema privileges
\c mydb
GRANT ALL ON SCHEMA public TO myuser;

-- Exit PostgreSQL
\q
```

### 6.4 Exit PostgreSQL User Session

```bash
# Exit back to ubuntu user
exit
```

---

# Step 7: Create Test Table and Data

### 7.1 Connect as New User

```bash
# Connect to mydb using myuser
sudo -u postgres psql -d mydb -U myuser -W
```

Enter the password (`StrongPassword123!`) when prompted.

### 7.2 Create Sample Table

```sql
-- Create a simple employees table
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    department VARCHAR(50),
    salary DECIMAL(10,2),
    hire_date DATE DEFAULT CURRENT_DATE
);

-- Create an index on last_name for better query performance
CREATE INDEX idx_employees_last_name ON employees(last_name);

-- Verify table creation
\d employees
```

### 7.3 Insert Sample Data

```sql
-- Insert sample records
INSERT INTO employees (first_name, last_name, email, department, salary)
VALUES 
    ('John', 'Doe', 'john.doe@example.com', 'Engineering', 85000.00),
    ('Jane', 'Smith', 'jane.smith@example.com', 'Marketing', 72000.00),
    ('Bob', 'Johnson', 'bob.johnson@example.com', 'Sales', 68000.00),
    ('Alice', 'Williams', 'alice.williams@example.com', 'Engineering', 92000.00),
    ('Charlie', 'Brown', 'charlie.brown@example.com', 'Marketing', 71000.00);

-- Verify data
SELECT * FROM employees;
```

### 7.4 Test Basic Queries

```sql
-- Count employees by department
SELECT department, COUNT(*) as employee_count, AVG(salary) as avg_salary
FROM employees
GROUP BY department
ORDER BY avg_salary DESC;

-- Find highest paid employee
SELECT first_name, last_name, salary
FROM employees
WHERE salary = (SELECT MAX(salary) FROM employees);

-- Exit PostgreSQL
\q
```

---

# Step 8: Configure PostgreSQL for Remote Access

### 8.1 Locate PostgreSQL Configuration Files

```bash
# Check where PostgreSQL configuration files are located
sudo -u postgres psql -c "SHOW config_file;"
sudo -u postgres psql -c "SHOW hba_file;"

# Typically located at:
# /etc/postgresql/16/main/postgresql.conf
# /etc/postgresql/16/main/pg_hba.conf
```

### 8.2 Edit postgresql.conf

```bash
# Open the configuration file (adjust version number as needed)
sudo nano /etc/postgresql/16/main/postgresql.conf

# Or if you prefer vi:
# sudo vi /etc/postgresql/16/main/postgresql.conf
```

Find and modify the following lines:

```conf
# Listen on all network interfaces (was: #listen_addresses = 'localhost')
listen_addresses = '*'

# Optional: Increase connection limit (default is 100)
max_connections = 200

# Optional: Increase shared buffers for better performance (25% of RAM)
shared_buffers = 128MB  # Adjust based on available memory
```

**To locate these lines**:
- Press `Ctrl+W` in nano to search
- Type `listen_addresses` and press Enter
- Uncomment the line (remove `#`) and set value to `'*'`

Save and exit:
- **nano**: `Ctrl+X`, then `Y`, then `Enter`
- **vi**: `Esc`, then `:wq`, then `Enter`

### 8.3 Edit pg_hba.conf

```bash
# Open the client authentication configuration
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

Add the following line at the **end** of the file:

```conf
# Allow remote connections from any IP (development only)
host    all             all             0.0.0.0/0               md5

# For better security, restrict to specific IP or subnet:
# host    mydb    myuser    192.168.1.0/24    md5
```

> **WARNING**: The `0.0.0.0/0` setting allows connections from ANY IP address. For production, specify your IP or use a more restrictive rule.

### 8.4 Restart PostgreSQL

```bash
# Restart PostgreSQL to apply changes
sudo systemctl restart postgresql

# Verify it's running
sudo systemctl status postgresql
```

### 8.5 Configure Firewall (UFW)

```bash
# Check UFW status
sudo ufw status

# If UFW is inactive, enable it (be careful - this may disconnect your SSH)
# First, allow SSH to avoid lockout
sudo ufw allow 22/tcp

# Allow PostgreSQL port
sudo ufw allow 5432/tcp

# Enable firewall
sudo ufw --force enable

# Check rules
sudo ufw status numbered
```

### 8.6 Verify PostgreSQL is Listening on All Interfaces

```bash
# Check which interfaces PostgreSQL is listening on
sudo netstat -tlnp | grep postgres

# Should show:
# tcp        0      0 0.0.0.0:5432            0.0.0.0:*               LISTEN      1234/postgres
# tcp6       0      0 :::5432                 :::*                    LISTEN      1234/postgres

# Alternative with ss command
sudo ss -tlnp | grep postgres
```

---

# Step 9: Test Local Connection

### 9.1 Test Connection from EC2 Locally

```bash
# Connect using localhost
psql -h localhost -U myuser -d mydb -W

# Connect using public IP (from within EC2)
psql -h $(curl -s http://checkip.amazonaws.com) -U myuser -d mydb -W

# Connect using hostname
psql -h postgresql-server -U myuser -d mydb -W
```

### 9.2 Test Simple Query

```sql
-- Should show your data
SELECT * FROM employees;

\q
```

---

# Step 10: Connect via pgAdmin

### 10.1 Launch pgAdmin

1. Open pgAdmin on your local machine
2. You'll see the dashboard with server groups

### 10.2 Create New Server Connection

1. Right-click on **Servers** in the left browser
2. Select **Register** → **Server**

### 10.3 Configure General Tab

| Field | Value |
|-------|-------|
| Name | `AWS PostgreSQL` (or any descriptive name) |

### 10.4 Configure Connection Tab

| Field | Value |
|-------|-------|
| Host name/address | `<EC2_PUBLIC_IP>` (e.g., 54.123.45.67) |
| Port | `5432` |
| Maintenance database | `mydb` |
| Username | `myuser` |
| Password | `StrongPassword123!` |
| Save password | ✓ Check this box |

### 10.5 Configure SSL Tab (Optional)

For a basic connection, SSL can be left as default. For production, you should configure SSL certificates.

### 10.6 Test Connection

1. Click **Save**
2. pgAdmin will attempt to connect
3. If successful, you'll see your server in the browser with:
   - Databases
   - Tables
   - etc.

### 10.7 Explore Your Database in pgAdmin

- Navigate to **Servers** → **AWS PostgreSQL** → **Databases** → **mydb** → **Schemas** → **public** → **Tables**
- Right-click on `employees` and select:
  - **View/Edit Data** → **All Rows** to see your data
  - **Properties** to see table structure
  - **Scripts** to generate SQL

---

# Step 11: Troubleshooting Common Issues

### 11.1 Connection Refused

**Error**: `could not connect to server: Connection refused`

**Solutions**:
```bash
# Check if PostgreSQL is running
sudo systemctl status postgresql

# Check if port is open
sudo netstat -tlnp | grep 5432

# Check firewall
sudo ufw status

# Check AWS Security Group - ensure 5432 is open to your IP
```

### 11.2 Authentication Failed

**Error**: `FATAL: password authentication failed for user "myuser"`

**Solutions**:
```bash
# Reset user password
sudo -u postgres psql -c "ALTER USER myuser WITH PASSWORD 'NewStrongPassword123!';"

# Check pg_hba.conf for correct authentication method
sudo cat /etc/postgresql/16/main/pg_hba.conf | grep -v "^#"
```

### 11.3 No pg_hba.conf Entry

**Error**: `FATAL: no pg_hba.conf entry for host`

**Solution**:
```bash
# Edit pg_hba.conf and add appropriate entry
sudo nano /etc/postgresql/16/main/pg_hba.conf
# Add: host    mydb    myuser    0.0.0.0/0    md5
sudo systemctl restart postgresql
```

### 11.4 Cannot Connect from pgAdmin

**Checklist**:
- [ ] EC2 instance is running
- [ ] Security Group allows inbound TCP 5432 from your IP
- [ ] PostgreSQL is listening on 0.0.0.0 (all interfaces)
- [ ] pg_hba.conf has entry for remote connections
- [ ] Firewall (UFW) allows port 5432
- [ ] Correct username/password
- [ ] EC2 public IP hasn't changed (if using a public IP, consider Elastic IP)

---

# Step 12: Performance Optimization (Optional)

### 12.1 Configure PostgreSQL for Performance

```bash
# Edit postgresql.conf
sudo nano /etc/postgresql/16/main/postgresql.conf
```

Recommended settings for a t3.xlarge (4 vCPU, 16GB RAM):

```conf
# Memory Settings
shared_buffers = 4GB              # 25% of RAM
effective_cache_size = 12GB       # 75% of RAM
work_mem = 64MB                   # Adjust based on concurrent queries
maintenance_work_mem = 1GB        # For maintenance operations

# Checkpoint Settings
checkpoint_completion_target = 0.9
checkpoint_timeout = 15min
max_wal_size = 4GB
min_wal_size = 1GB

# Query Planning
random_page_cost = 1.1            # For SSD storage
effective_io_concurrency = 200    # For SSD

# Connection Settings
max_connections = 200
```

### 12.2 Install pg_stat_statements (Query Monitoring)

```bash
# Enable extension
sudo -u postgres psql -d mydb -c "CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"

# Edit postgresql.conf to preload
sudo nano /etc/postgresql/16/main/postgresql.conf
```

Add or uncomment:
```conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

Restart PostgreSQL:
```bash
sudo systemctl restart postgresql
```

---

# Step 13: Backup and Maintenance

### 13.1 Backup Database

```bash
# Backup entire database
pg_dump -U myuser -h localhost mydb > mydb_backup_$(date +%Y%m%d).sql

# Compress backup
pg_dump -U myuser -h localhost mydb | gzip > mydb_backup_$(date +%Y%m%d).sql.gz

# Backup all databases
pg_dumpall -U postgres > all_databases_backup.sql
```

### 13.2 Restore Database

```bash
# Restore from backup
psql -U myuser -d mydb < mydb_backup_20240101.sql

# Restore from compressed backup
gunzip -c mydb_backup_20240101.sql.gz | psql -U myuser -d mydb
```

### 13.3 Set Up Automated Backups with Cron

```bash
# Edit crontab for postgres user
sudo crontab -u postgres -e

# Add daily backup at 2 AM
0 2 * * * pg_dump -U postgres mydb > /home/ubuntu/backups/mydb_$(date +\%Y\%m\%d).sql
```

---

# Step 14: Security Best Practices

### 14.1 Change Default PostgreSQL Port (Optional)

```bash
# Edit postgresql.conf
sudo nano /etc/postgresql/16/main/postgresql.conf

# Change port
port = 5433

# Update firewall
sudo ufw allow 5433/tcp

# Update security group in AWS console to allow new port
# Restart PostgreSQL
sudo systemctl restart postgresql
```

### 14.2 Enable SSL/TLS

```bash
# Generate self-signed certificate (development only)
sudo openssl req -new -text -nodes -subj "/CN=postgresql-server" -out server.req
sudo openssl rsa -in privkey.pem -out server.key
sudo openssl req -x509 -in server.req -text -key server.key -out server.crt
sudo chmod 600 server.key
sudo mv server.key server.crt /etc/postgresql/16/main/

# Configure SSL in postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
```

### 14.3 Create Read-Only User

```sql
-- Create read-only user for reporting
CREATE USER readonly WITH PASSWORD 'ReadOnly123!';
GRANT CONNECT ON DATABASE mydb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;
```

### 14.4 Enable Query Logging

```conf
# In postgresql.conf
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_statement = 'ddl'  # Log DDL statements
log_min_duration_statement = 1000  # Log queries slower than 1 second
```

---

# Step 15: Clean Up (When Done)

### 15.1 Stop PostgreSQL

```bash
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

### 15.2 Terminate EC2 Instance

1. Go to AWS EC2 Console → Instances
2. Select your `postgresql-server` instance
3. Click **Instance State** → **Terminate**
4. Confirm termination

> **Warning**: Terminating an instance permanently deletes it. Ensure you have backups if needed.

### 15.3 Delete Security Group

1. Go to EC2 Console → Security Groups
2. Find `postgresql-sg`
3. Click **Actions** → **Delete**

### 15.4 Delete Key Pair

1. Go to EC2 Console → Key Pairs
2. Find `postgresql-server-key`
3. Click **Actions** → **Delete**
4. Also delete the local `.pem` file from your computer

---

## Useful Commands Reference

```bash
# PostgreSQL Service
sudo systemctl status|start|stop|restart postgresql

# Connect as postgres user
sudo -i -u postgres
psql

# Connect to specific database
psql -h localhost -U myuser -d mydb

# List databases
psql -l

# Export/Import
pg_dump -U myuser mydb > backup.sql
psql -U myuser -d mydb < backup.sql

# Show PostgreSQL version
psql --version

# Show active connections
sudo -u postgres psql -c "SELECT * FROM pg_stat_activity;"
```

---

## Additional Resources

- [PostgreSQL Official Documentation](https://www.postgresql.org/docs/)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [pgAdmin Documentation](https://www.pgadmin.org/docs/)
- [PostgreSQL Performance Tuning](https://wiki.postgresql.org/wiki/Performance_Optimization)

---

*This guide was created for educational purposes. Always follow security best practices for production deployments.*
