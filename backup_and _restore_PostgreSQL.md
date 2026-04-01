# PostgreSQL on EC2: Connection Testing, Backup, and Restoration Guide

## 1.0 Overview

This document provides a comprehensive guide for:
- Testing connectivity to a publicly hosted PostgreSQL database on an EC2 instance.
- Establishing an SSH connection to the EC2 server.
- Verifying PostgreSQL service status and connectivity.
- Performing a database backup.
- Restoring a database from a backup file.

**Prerequisites:**
- Access to the EC2 instance’s public IP address.
- The PostgreSQL database credentials.
- The private key (`.pem` file) for SSH access.
- `psql` and `pg_dump` command-line tools installed locally or on the EC2 instance.
- Proper security group rules allowing access to ports `22` (SSH) and `5432` (PostgreSQL) from your IP address.

---

## 2.0 Testing Database Connectivity

### 2.1 Test Remote Connection (from your local machine)

Use `psql` or a database client to test the connection to the publicly accessible PostgreSQL instance.

```bash
# Test using psql with the provided DATABASE_URL
psql "postgresql://postgres:password@<EC2_PUBLIC_IP>:5432/db_name"

# Or use the interactive prompt with password
psql -h <EC2_PUBLIC_IP> -p 5432 -U postgres -d db_name
```

**Expected Output:**  
If successful, you will enter the PostgreSQL interactive terminal. Type `\q` to exit.

**Troubleshooting:**
- **Connection timeout:** Check security group inbound rules.
- **Authentication failed:** Verify username and password.
- **Database not found:** Confirm database name.

---

## 3.0 Establishing SSH Connection

### 3.1 Connect via SSH

Use the private key to connect to the EC2 instance. Ensure the key file has correct permissions.

```bash
# Set correct permissions for the key file (if not already done)
chmod 400 postgresql-server-key.pem

# Connect to the EC2 instance
ssh -i "postgresql-server-key.pem" ubuntu@<EC2_PUBLIC_IP>
```

**Note:** Replace `ubuntu` with the appropriate username for your EC2 AMI (e.g., `ec2-user` for Amazon Linux).

### 3.2 Verify PostgreSQL Service Status

Once connected, check if PostgreSQL is running.

```bash
# Check PostgreSQL service status
sudo systemctl status postgresql

# If using Amazon Linux/RHEL, the service name might be 'postgresql'
sudo systemctl status postgresql-14   # Adjust version as needed
```

**Expected Output:**  
Active (running) status.

### 3.3 Test Local PostgreSQL Connection

Switch to the `postgres` system user and test local connectivity.

```bash
# Switch to postgres user
sudo -i -u postgres

# Connect to PostgreSQL
psql -d db_name
```

Run a test query:

```sql
SELECT version();
SELECT now();
\q
```

---

## 4.0 Database Backup

Backups can be performed either locally (after SSH) or remotely. We'll cover both methods.

### 4.1 Backup via SSH (on EC2 Instance)

This method uses `pg_dump` on the server and saves the backup locally on the EC2 instance.

#### 4.1.1 Full Database Backup

```bash
# SSH into EC2
ssh -i "postgresql-server-key.pem" ubuntu@<EC2_PUBLIC_IP>

# Switch to postgres user and perform backup
sudo -i -u postgres

# Dump the entire database
pg_dump -d db_name -U postgres -F c -f /tmp/db_name_backup.dump
```

**Explanation:**
- `-F c`: Custom format (compressed, allows parallel restore).
- `-f`: Output file path.

#### 4.1.2 Backup with Plain SQL Format

For portability, you can use plain SQL format:

```bash
pg_dump -d db_name -U postgres -f /tmp/db_name_backup.sql
```

#### 4.1.3 Transfer Backup to Local Machine

After backup, copy the file to your local system:

```bash
# On local machine, from another terminal
scp -i "postgresql-server-key.pem" ubuntu@<EC2_PUBLIC_IP>:/tmp/db_name_backup.dump ./
```

### 4.2 Remote Backup (from Local Machine)

You can also run `pg_dump` from your local machine without SSH into EC2.

```bash
# Using connection string
pg_dump "postgresql://postgres:password@<EC2_PUBLIC_IP>:5432/db_name" -F c -f local_backup.dump

# Using host and database parameters
pg_dump -h <EC2_PUBLIC_IP> -p 5432 -U postgres -d db_name -F c -f local_backup.dump
```

---

## 5.0 Database Restoration

Restoration can be performed on the same or a different database. We'll cover restoring to the existing database and creating a new one.

### 5.1 Restore from Backup (On EC2)

#### 5.1.1 Restore Custom Format Backup

```bash
# SSH into EC2
ssh -i "postgresql-server-key.pem" ubuntu@<EC2_PUBLIC_IP>

# Switch to postgres user
sudo -i -u postgres

# Drop the existing database if restoring over it
dropdb -U postgres db_name

# Create a fresh database
createdb -U postgres db_name

# Restore from custom format backup
pg_restore -U postgres -d db_name -v /tmp/db_name_backup.dump
```

#### 5.1.2 Restore Plain SQL Backup

```bash
# Connect to database and execute SQL script
psql -U postgres -d db_name -f /tmp/db_name_backup.sql
```

### 5.2 Restore from Local Machine

#### 5.2.1 Restore Custom Format from Local

```bash
# First, optionally drop and recreate the database remotely
psql "postgresql://postgres:password@<EC2_PUBLIC_IP>:5432/postgres" -c "DROP DATABASE IF EXISTS db_name;"
psql "postgresql://postgres:password@<EC2_PUBLIC_IP>:5432/postgres" -c "CREATE DATABASE db_name;"

# Restore from local backup file
pg_restore -h <EC2_PUBLIC_IP> -p 5432 -U postgres -d db_name -v local_backup.dump
```

### 5.3 Restore to a Different Database

If you want to restore to a new database (e.g., `db_name_restored`):

```bash
# Create the new database
psql "postgresql://postgres:password@<EC2_PUBLIC_IP>:5432/postgres" -c "CREATE DATABASE db_name_restored;"

# Restore into the new database
pg_restore -h <EC2_PUBLIC_IP> -p 5432 -U postgres -d db_name_restored -v local_backup.dump
```

---

## 6.0 Verification and Testing

### 6.1 Verify Restored Data

Connect to the restored database and run verification queries:

```bash
psql "postgresql://postgres:password@<EC2_PUBLIC_IP>:5432/db_name"
```

```sql
-- Check all tables
\dt

-- Count rows in a specific table (adjust table name)
SELECT COUNT(*) FROM your_table;

-- Check recent data
SELECT * FROM your_table LIMIT 10;
```

### 6.2 Compare Row Counts

If you have access to original database, compare row counts:

```sql
-- On original database
SELECT schemaname, tablename, n_live_tup 
FROM pg_stat_user_tables 
ORDER BY n_live_tup DESC;

-- On restored database (same query)
```

---

## 7.0 Troubleshooting

| Issue | Solution |
|-------|----------|
| `psql: could not connect to server: Connection timed out` | Check EC2 security group inbound rules for port 5432. |
| `FATAL: password authentication failed` | Verify password. Check `pg_hba.conf` for proper authentication method. |
| `pg_dump: [archiver] connection to database failed` | Ensure PostgreSQL is running and credentials are correct. |
| `permission denied` for key file | Run `chmod 400 postgresql-server-key.pem`. |
| `role "postgres" does not exist` | Use correct username; sometimes the default is `postgres` or `ec2-user`. |
| `database "db_name" does not exist` | Create the database before restoring: `CREATE DATABASE db_name;`. |

---

## 8.0 Best Practices

1. **Always test backups** by restoring to a non-production environment first.
2. **Use custom format** (`-F c`) for flexibility and compression.
3. **Secure your backups** – encrypt sensitive data if necessary.
4. **Automate backups** using cron jobs or scripts.
5. **Monitor disk space** on EC2 to ensure backup files don't fill the disk.
6. **Use `pg_dumpall`** for backing up all databases and global objects.
7. **Consider logical vs physical backups** – logical (`pg_dump`) is portable, physical (file-level) is faster for large databases.

---

## 9.0 Appendix: Quick Reference Commands

### Connection Test
```bash
psql "postgresql://postgres:password@<PUBLIC_IP>:5432/db_name"
```

### SSH
```bash
ssh -i "postgresql-server-key.pem" ubuntu@<PUBLIC_IP>
```

### Backup (Custom Format)
```bash
pg_dump -h <PUBLIC_IP> -U postgres -d db_name -F c -f backup.dump
```

### Restore (Custom Format)
```bash
pg_restore -h <PUBLIC_IP> -U postgres -d db_name -v backup.dump
```

### Drop and Recreate Database
```sql
DROP DATABASE IF EXISTS db_name;
CREATE DATABASE db_name;
```

---
