# Cron Job Automation Notes

## Table of Contents
1. [Introduction to Cron](#introduction-to-cron)
2. [Cron Basics](#cron-basics)
3. [Crontab Syntax](#crontab-syntax)
4. [Special Characters](#special-characters)
5. [Common Examples](#common-examples)
6. [Environment Variables](#environment-variables)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)
9. [Advanced Topics](#advanced-topics)

## Introduction to Cron

Cron is a time-based job scheduler in Unix-like operating systems. It enables users to schedule jobs (commands or scripts) to run periodically at fixed times, dates, or intervals.

### Key Components
- **cron daemon (crond)**: The background service that executes scheduled commands
- **crontab**: The file where cron jobs are defined
- **cron expression**: The syntax used to define when a job should run

## Cron Basics

### Managing Crontab

```bash
# View current user's crontab
crontab -l

# Edit current user's crontab
crontab -e

# Remove current user's crontab
crontab -r

# Edit another user's crontab (requires privileges)
sudo crontab -u username -e

# View system-wide cron jobs
ls -la /etc/cron.*
cat /etc/crontab
```

### Cron Service Management

```bash
# Check cron service status
sudo systemctl status cron    # Debian/Ubuntu
sudo systemctl status crond   # RHEL/CentOS

# Start/Stop/Restart cron service
sudo systemctl start cron
sudo systemctl stop cron
sudo systemctl restart cron

# Enable cron to start at boot
sudo systemctl enable cron
```

## Crontab Syntax

### Basic Format

```
* * * * * command_to_execute
│ │ │ │ │
│ │ │ │ └─── Day of the Week (0-7, 0 and 7 = Sunday)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of the Month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)
```

### Field Values

| Field | Allowed Values | Special Characters |
|-------|---------------|-------------------|
| Minute | 0-59 | * , - / |
| Hour | 0-23 | * , - / |
| Day of Month | 1-31 | * , - / ? L W |
| Month | 1-12 or JAN-DEC | * , - / |
| Day of Week | 0-7 or SUN-SAT | * , - / ? L # |

## Special Characters

- **`*`** - Matches any value (wildcard)
- **`,`** - Separates multiple values
- **`-`** - Defines a range
- **`/`** - Defines step values
- **`?`** - No specific value (used in day fields)
- **`L`** - Last (e.g., L in day of month = last day)
- **`W`** - Weekday (nearest weekday)
- **`#`** - Nth occurrence (e.g., 2#3 = 3rd Tuesday)

### Special Strings (Shortcuts)

```bash
@reboot     # Run once at startup
@yearly     # Run once a year (0 0 1 1 *)
@annually   # Same as @yearly
@monthly    # Run once a month (0 0 1 * *)
@weekly     # Run once a week (0 0 * * 0)
@daily      # Run once a day (0 0 * * *)
@midnight   # Same as @daily
@hourly     # Run once an hour (0 * * * *)
```

## Common Examples

### Basic Examples

```bash
# Run every minute
* * * * * /path/to/script.sh

# Run every 5 minutes
*/5 * * * * /path/to/script.sh

# Run at 2:30 AM every day
30 2 * * * /path/to/script.sh

# Run at 6 PM Monday through Friday
0 18 * * 1-5 /path/to/script.sh

# Run at midnight on the first day of every month
0 0 1 * * /path/to/script.sh

# Run every Sunday at 5 PM
0 17 * * 0 /path/to/script.sh

# Run on specific dates (1st and 15th) at 3:30 PM
30 15 1,15 * * /path/to/script.sh

# Run every 2 hours
0 */2 * * * /path/to/script.sh

# Run at system startup
@reboot /path/to/startup_script.sh
```

### Real-World Examples

```bash
# Backup database every day at 2 AM
0 2 * * * /usr/local/bin/backup_database.sh

# Clean up log files every week
0 3 * * 0 find /var/log -name "*.log" -mtime +7 -delete

# Update system packages weekly
0 1 * * 0 apt update && apt upgrade -y

# Monitor disk space every hour
0 * * * * df -h | mail -s "Disk Usage Report" admin@example.com

# Sync files to remote server every 30 minutes
*/30 * * * * rsync -avz /local/path/ user@remote:/remote/path/

# Generate reports on weekdays at 9 AM
0 9 * * 1-5 /opt/scripts/generate_daily_report.py

# Clear cache every night at midnight
0 0 * * * rm -rf /var/cache/myapp/*

# Check website availability every 10 minutes
*/10 * * * * curl -f https://example.com || echo "Site is down" | mail -s "Alert" admin@example.com
```

## Environment Variables

### Setting Environment Variables in Crontab

```bash
# Set PATH
PATH=/usr/local/bin:/usr/bin:/bin

# Set shell
SHELL=/bin/bash

# Set email for notifications
MAILTO=admin@example.com

# Set home directory
HOME=/home/username

# Custom variables
DATABASE_URL=postgresql://localhost/mydb
API_KEY=your_secret_key

# Example job using environment variables
0 2 * * * $HOME/scripts/backup.sh >> $HOME/logs/backup.log 2>&1
```

### Loading Environment from File

```bash
# Source environment file before running command
0 * * * * . /home/user/.env && /path/to/script.sh
```

## Best Practices

### 1. Use Absolute Paths

```bash
# Good
0 * * * * /usr/local/bin/python3 /home/user/scripts/task.py

# Bad
0 * * * * python3 scripts/task.py
```

### 2. Redirect Output

```bash
# Redirect both stdout and stderr to log file
0 * * * * /path/to/script.sh >> /var/log/cron_script.log 2>&1

# Discard output
0 * * * * /path/to/script.sh > /dev/null 2>&1

# Separate error and output logs
0 * * * * /path/to/script.sh >> /var/log/output.log 2>> /var/log/error.log
```

### 3. Add Comments

```bash
# Daily backup of production database
0 2 * * * /usr/local/bin/backup_prod_db.sh

# Weekly system maintenance
0 3 * * 0 /usr/local/bin/system_maintenance.sh
```

### 4. Test Before Scheduling

```bash
# Test your command manually first
/path/to/script.sh

# Test with cron environment
env -i SHELL=/bin/sh PATH=/usr/bin:/bin /path/to/script.sh
```

### 5. Use Lock Files to Prevent Overlaps

```bash
# In your script
LOCKFILE=/var/run/myscript.lock

if [ -e ${LOCKFILE} ] && kill -0 `cat ${LOCKFILE}`; then
    echo "Script already running"
    exit
fi

echo $$ > ${LOCKFILE}
# Your script logic here
rm -f ${LOCKFILE}
```

### 6. Log Execution Time

```bash
0 * * * * echo "Job started at $(date)" >> /var/log/job.log && /path/to/script.sh >> /var/log/job.log 2>&1
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Job Not Running

```bash
# Check if cron service is running
systemctl status cron

# Check cron logs
# Ubuntu/Debian
grep CRON /var/log/syslog

# RHEL/CentOS
tail -f /var/log/cron

# Check for syntax errors
crontab -l
```

#### 2. Permission Issues

```bash
# Ensure script is executable
chmod +x /path/to/script.sh

# Check file ownership
ls -la /path/to/script.sh

# Run with specific user permissions
0 * * * * sudo -u www-data /path/to/script.sh
```

#### 3. Environment Issues

```bash
# Debug environment differences
* * * * * env > /tmp/cron_env.txt
# Then compare with your shell environment
```

#### 4. Path Issues

```bash
# Always use full paths or set PATH in crontab
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 * * * * my_command
```

### Debugging Commands

```bash
# Test cron expression
# Online tool: crontab.guru

# Monitor cron execution in real-time
tail -f /var/log/syslog | grep CRON

# List all user crontabs
for user in $(cut -f1 -d: /etc/passwd); do 
    echo "Crontab for $user:"
    sudo crontab -u $user -l 2>/dev/null
done

# Check mail for cron errors
mail
```

## Advanced Topics

### 1. Running Jobs with Specific Time Zones

```bash
# Set timezone for specific job
CRON_TZ=America/New_York
0 9 * * * /path/to/script.sh

# Or in the script itself
export TZ=America/New_York
```

### 2. Conditional Execution

```bash
# Run only if file exists
* * * * * [ -f /path/to/trigger.file ] && /path/to/script.sh

# Run based on system load
* * * * * [ $(cat /proc/loadavg | cut -d' ' -f1 | cut -d'.' -f1) -lt 2 ] && /path/to/heavy_task.sh

# Run on specific host
* * * * * [ $(hostname) = "prod-server" ] && /path/to/prod_task.sh
```

### 3. Using Anacron for Laptops/Desktops

```bash
# Install anacron
sudo apt install anacron

# Anacron configuration (/etc/anacrontab)
# period  delay  job-id  command
1         5      daily-backup    /usr/local/bin/daily_backup.sh
7         10     weekly-cleanup  /usr/local/bin/weekly_cleanup.sh
```

### 4. Distributed Cron with Job Queues

```bash
# Using at command for one-time jobs
echo "/path/to/script.sh" | at now + 2 hours
echo "/path/to/script.sh" | at 10:00 PM tomorrow

# List queued jobs
atq

# Remove job
atrm job_number
```

### 5. Monitoring Cron Jobs

```bash
# Create a wrapper script for monitoring
#!/bin/bash
JOB_NAME="$1"
shift
START_TIME=$(date +%s)

# Run the actual command
"$@"
EXIT_CODE=$?

END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

# Log to monitoring system
echo "Job: $JOB_NAME, Duration: $DURATION, Exit Code: $EXIT_CODE" >> /var/log/cron_monitor.log

# Send alert if failed
if [ $EXIT_CODE -ne 0 ]; then
    echo "Cron job $JOB_NAME failed" | mail -s "Cron Alert" admin@example.com
fi

exit $EXIT_CODE

# Use in crontab
0 * * * * /usr/local/bin/cron_wrapper.sh "hourly_task" /path/to/actual_script.sh
```

### 6. Security Considerations

```bash
# Restrict cron access
# /etc/cron.allow - Users allowed to use cron
# /etc/cron.deny - Users denied from using cron

# Set secure permissions on scripts
chmod 700 /path/to/script.sh
chown root:root /path/to/script.sh

# Avoid putting sensitive data in crontab
# Use external config files instead
0 * * * * /path/to/script.sh --config /etc/myapp/secure.conf
```

## Example Automation Scripts

### 1. Database Backup Script

```bash
#!/bin/bash
# /usr/local/bin/backup_database.sh

BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
DB_USER="backup_user"
DB_PASS="secure_password"
DB_NAME="production"

# Create backup
mysqldump -u$DB_USER -p$DB_PASS $DB_NAME | gzip > $BACKUP_DIR/backup_$DATE.sql.gz

# Remove backups older than 7 days
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +7 -delete

# Cron entry
# 0 2 * * * /usr/local/bin/backup_database.sh >> /var/log/backup.log 2>&1
```

### 2. System Health Check

```bash
#!/bin/bash
# /usr/local/bin/health_check.sh

THRESHOLD=90
EMAIL="admin@example.com"

# Check disk usage
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt $THRESHOLD ]; then
    echo "Disk usage is at ${DISK_USAGE}%" | mail -s "Disk Alert" $EMAIL
fi

# Check memory usage
MEM_USAGE=$(free | awk 'NR==2{printf "%.0f", $3*100/$2}')
if [ $MEM_USAGE -gt $THRESHOLD ]; then
    echo "Memory usage is at ${MEM_USAGE}%" | mail -s "Memory Alert" $EMAIL
fi

# Check load average
LOAD_AVG=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | sed 's/,//')
CPU_COUNT=$(nproc)
if (( $(echo "$LOAD_AVG > $CPU_COUNT" | bc -l) )); then
    echo "High load average: $LOAD_AVG" | mail -s "Load Alert" $EMAIL
fi

# Cron entry
# */15 * * * * /usr/local/bin/health_check.sh
```

### 3. Log Rotation Script

```bash
#!/bin/bash
# /usr/local/bin/rotate_logs.sh

LOG_DIR="/var/log/myapp"
ARCHIVE_DIR="/var/log/myapp/archive"
DAYS_TO_KEEP=30

# Create archive directory if it doesn't exist
mkdir -p $ARCHIVE_DIR

# Rotate logs
for log in $LOG_DIR/*.log; do
    if [ -f "$log" ]; then
        filename=$(basename "$log")
        mv "$log" "$ARCHIVE_DIR/${filename}.$(date +%Y%m%d)"
        touch "$log"
        chmod 644 "$log"
    fi
done

# Compress old logs
find $ARCHIVE_DIR -name "*.log.*" -not -name "*.gz" -exec gzip {} \;

# Delete old compressed logs
find $ARCHIVE_DIR -name "*.gz" -mtime +$DAYS_TO_KEEP -delete

# Cron entry
# 0 0 * * * /usr/local/bin/rotate_logs.sh
```

## Quick Reference Card

```bash
# Every minute
* * * * * command

# Every hour at minute 0
0 * * * * command

# Every day at 2:30 AM
30 2 * * * command

# Every Monday at 8 PM
0 20 * * 1 command

# Every 15 minutes
*/15 * * * * command

# Every 2 hours between 8 AM and 6 PM on weekdays
0 8-18/2 * * 1-5 command

# First Monday of every month at 6 AM
0 6 1-7 * 1 command

# Last day of every month at 11:59 PM
59 23 L * * command

# Every quarter (Jan, Apr, Jul, Oct) on the 1st at midnight
0 0 1 1,4,7,10 * command
```

## Additional Resources

- [Crontab Generator](https://crontab.guru/) - Visual cron expression generator
- [Cron Expression Parser](https://cronexpressionparser.com/) - Parse and explain cron expressions
- `man crontab` - Manual page for crontab
- `man 5 crontab` - Manual page for crontab file format
- System logs: `/var/log/syslog` (Debian/Ubuntu) or `/var/log/cron` (RHEL/CentOS)

---

Remember: Always test your cron jobs thoroughly before deploying them to production, and monitor their execution regularly to ensure they're working as expected.
