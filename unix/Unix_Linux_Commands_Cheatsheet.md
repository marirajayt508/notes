# Unix/Linux Commands Cheatsheet - Amazon Lab126 Support Engineer

## Table of Contents
1. [Process Management](#process-management)
2. [File Operations](#file-operations)
3. [Disk Management](#disk-management)
4. [System Monitoring](#system-monitoring)
5. [Network Commands](#network-commands)
6. [Log Analysis](#log-analysis)
7. [Service Management](#service-management)
8. [Performance Analysis](#performance-analysis)
9. [Troubleshooting Commands](#troubleshooting-commands)
10. [Text Processing](#text-processing)
11. [System Information](#system-information)

---

## Process Management

### Process Viewing
```bash
# Basic process listing
ps                          # Current user processes
ps aux                      # All processes with details
ps -ef                      # Full format listing
ps -eo pid,ppid,cmd,etime   # Custom format with runtime

# Process tree
pstree                      # Process hierarchy
pstree -p                   # Include PIDs
pstree -u                   # Show user transitions

# Real-time monitoring
top                         # Interactive process monitor
htop                        # Enhanced top (if available)
top -p PID                  # Monitor specific process
top -u username             # Monitor user processes
```

### Process Control
```bash
# Kill processes
kill PID                    # Send SIGTERM (graceful)
kill -9 PID                 # Send SIGKILL (force)
kill -TERM PID              # Explicit SIGTERM
kill -HUP PID               # Send SIGHUP (reload config)

# Kill by name/pattern
killall process_name        # Kill all instances by name
pkill -f pattern           # Kill by command pattern
pkill -u username          # Kill all user processes

# Process search
pgrep process_name         # Find PID by name
pgrep -f pattern           # Find PID by command pattern
pgrep -l pattern           # Show PID and name
```

### Background Processes
```bash
# Job control
command &                   # Run in background
nohup command &            # Run immune to hangups
disown %1                  # Remove job from shell
jobs                       # List active jobs
fg %1                      # Bring job to foreground
bg %1                      # Send job to background

# Process priority
nice -n 10 command         # Run with lower priority
renice 5 PID              # Change priority of running process
```

---

## File Operations

### File Information
```bash
# File details
ls -la                     # Detailed listing with permissions
ls -lh                     # Human readable sizes
ls -lt                     # Sort by modification time
ls -lS                     # Sort by size
stat filename              # Detailed file statistics
file filename              # File type information
```

### File Permissions
```bash
# Change permissions
chmod 755 file             # rwxr-xr-x
chmod u+x file             # Add execute for owner
chmod g-w file             # Remove write for group
chmod o=r file             # Set read-only for others

# Change ownership
chown user:group file      # Change owner and group
chown user file            # Change owner only
chgrp group file           # Change group only
```

### File Operations
```bash
# Copy and move
cp source dest             # Copy file
cp -r source dest          # Copy directory recursively
cp -p source dest          # Preserve permissions/timestamps
mv source dest             # Move/rename file
rsync -av source dest      # Advanced sync with progress

# Links
ln file hardlink           # Create hard link
ln -s target softlink      # Create symbolic link
readlink linkname          # Show link target

# Find files
find /path -name "*.log"   # Find by name pattern
find /path -type f -size +100M  # Find large files
find /path -mtime -7       # Modified in last 7 days
find /path -user username  # Find by owner
locate filename            # Quick file search (if available)
which command              # Find command location
```

---

## Disk Management

### Disk Usage and Information
```bash
# Disk space
df -h                      # Human readable disk usage
df -i                      # Inode usage
du -sh /path               # Directory size summary
du -h --max-depth=1 /path  # Size of subdirectories
du -sh * | sort -rh        # Largest directories first

# Find large files
find / -size +100M -type f -exec ls -lh {} \; 2>/dev/null
find /var/log -name "*.log" -size +50M
ncdu /path                 # Interactive disk usage (if available)
```

### Disk Partitions and Mounting
```bash
# Partition information
lsblk                      # List block devices
lsblk -f                   # Show filesystem info
fdisk -l                   # List all partitions
parted -l                  # Alternative partition listing
blkid                      # Show UUID and filesystem types

# Mount operations
mount                      # Show mounted filesystems
mount /dev/sdb1 /mnt       # Mount partition
umount /mnt                # Unmount
umount -f /mnt             # Force unmount
mount -o remount,ro /path  # Remount as read-only

# Filesystem check and repair
fsck /dev/sdb1             # Check filesystem
fsck -y /dev/sdb1          # Auto-repair filesystem
e2fsck /dev/sdb1           # Check ext2/3/4 filesystem
xfs_repair /dev/sdb1       # Repair XFS filesystem
```

### LVM (Logical Volume Management)
```bash
# Physical volumes
pvdisplay                  # Show physical volumes
pvcreate /dev/sdb          # Create physical volume
pvremove /dev/sdb          # Remove physical volume

# Volume groups
vgdisplay                  # Show volume groups
vgcreate vg_name /dev/sdb  # Create volume group
vgextend vg_name /dev/sdc  # Add PV to VG
vgreduce vg_name /dev/sdc  # Remove PV from VG

# Logical volumes
lvdisplay                  # Show logical volumes
lvcreate -L 10G -n lv_name vg_name  # Create 10GB LV
lvextend -L +5G /dev/vg/lv # Extend LV by 5GB
lvreduce -L -2G /dev/vg/lv # Reduce LV by 2GB
resize2fs /dev/vg/lv       # Resize ext filesystem
```

### Disk I/O and Performance
```bash
# I/O monitoring
iostat -x 1 5              # Extended I/O stats every second
iotop                      # I/O usage by process
iotop -o                   # Only show processes doing I/O
sar -d 1 5                 # Disk activity statistics

# Disk performance testing
dd if=/dev/zero of=testfile bs=1M count=1000  # Write test
dd if=testfile of=/dev/null bs=1M             # Read test
hdparm -tT /dev/sda        # Disk speed test
badblocks -v /dev/sdb      # Check for bad blocks
```

### Filesystem Operations
```bash
# Create filesystems
mkfs.ext4 /dev/sdb1        # Create ext4 filesystem
mkfs.xfs /dev/sdb1         # Create XFS filesystem
mkfs.vfat /dev/sdb1        # Create FAT32 filesystem

# Filesystem information
tune2fs -l /dev/sdb1       # Show ext filesystem info
xfs_info /dev/sdb1         # Show XFS filesystem info
dumpe2fs /dev/sdb1         # Detailed ext filesystem info

# Filesystem maintenance
tune2fs -c 30 /dev/sdb1    # Set max mount count
tune2fs -i 180d /dev/sdb1  # Set check interval
xfs_fsr /dev/sdb1          # Defragment XFS filesystem
```

---

## System Monitoring

### CPU and Memory
```bash
# CPU monitoring
top                        # Real-time CPU usage
htop                       # Enhanced process viewer
vmstat 1 5                 # Virtual memory stats
mpstat 1 5                 # Multi-processor stats
sar -u 1 5                 # CPU utilization

# Memory monitoring
free -m                    # Memory usage in MB
free -h                    # Human readable memory
cat /proc/meminfo          # Detailed memory info
vmstat -s                  # Memory statistics
pmap PID                   # Process memory map
```

### Load and Performance
```bash
# System load
uptime                     # Load average and uptime
w                          # Who is logged in and load
cat /proc/loadavg          # Load average file

# Performance monitoring
sar -A                     # All system activity
pidstat -u 1 5             # Per-process CPU usage
pidstat -r 1 5             # Per-process memory usage
pidstat -d 1 5             # Per-process disk I/O
```

---

## Network Commands

### Network Configuration
```bash
# Interface information
ip addr show               # Show IP addresses
ip link show               # Show network interfaces
ifconfig                   # Interface configuration (legacy)
ip route show              # Show routing table
route -n                   # Numeric routing table (legacy)

# Network statistics
netstat -tuln              # TCP/UDP listening ports
netstat -rn                # Routing table
ss -tuln                   # Socket statistics (modern)
ss -tulpn                  # Include process names
```

### Network Connectivity
```bash
# Basic connectivity
ping -c 4 host             # Ping with count
ping6 -c 4 host            # IPv6 ping
traceroute host            # Trace route to host
tracepath host             # Alternative traceroute
mtr host                   # Continuous traceroute

# DNS and resolution
nslookup domain            # DNS lookup
dig domain                 # Detailed DNS query
dig @8.8.8.8 domain       # Query specific DNS server
host domain                # Simple DNS lookup
```

### Network Troubleshooting
```bash
# Port testing
telnet host port           # Test TCP connection
nc -zv host port           # Test port with netcat
nmap -p port host          # Port scan

# Network monitoring
tcpdump -i eth0            # Packet capture
tcpdump -i eth0 port 80    # Capture HTTP traffic
wireshark                  # GUI packet analyzer
iftop                      # Network bandwidth usage
nethogs                   # Network usage by process
```

---

## Log Analysis

### Log File Locations
```bash
# Common log locations
/var/log/syslog            # System messages
/var/log/messages          # General messages
/var/log/auth.log          # Authentication logs
/var/log/kern.log          # Kernel messages
/var/log/dmesg             # Boot messages
/var/log/cron.log          # Cron job logs
```

### Log Viewing Commands
```bash
# Basic log viewing
tail -f /var/log/syslog    # Follow log in real-time
tail -n 100 /var/log/syslog # Last 100 lines
head -n 50 /var/log/syslog  # First 50 lines
less /var/log/syslog       # Page through log
zless /var/log/syslog.gz   # View compressed logs

# Multiple files
multitail /var/log/syslog /var/log/auth.log  # Multiple logs
tail -f /var/log/*.log     # Follow all logs
```

### Log Analysis
```bash
# Search logs
grep "ERROR" /var/log/syslog           # Find errors
grep -i "failed" /var/log/auth.log     # Case insensitive
grep -r "pattern" /var/log/            # Recursive search
zgrep "pattern" /var/log/*.gz          # Search compressed logs

# Log statistics
grep -c "ERROR" /var/log/syslog        # Count occurrences
awk '/ERROR/ {print $1, $2, $3}' /var/log/syslog  # Extract fields
```

### Systemd Logs (journalctl)
```bash
# Basic journal commands
journalctl                 # All journal entries
journalctl -f              # Follow journal
journalctl -u service      # Specific service logs
journalctl --since "1 hour ago"  # Recent entries
journalctl --until "2023-01-01"  # Entries until date

# Journal filtering
journalctl -p err          # Error priority and above
journalctl -k              # Kernel messages only
journalctl _PID=1234       # Specific process
journalctl --disk-usage    # Journal disk usage
```

---

## Service Management

### Systemd Services
```bash
# Service control
systemctl start service    # Start service
systemctl stop service     # Stop service
systemctl restart service  # Restart service
systemctl reload service   # Reload configuration
systemctl enable service   # Enable at boot
systemctl disable service  # Disable at boot

# Service status
systemctl status service   # Service status
systemctl is-active service # Check if active
systemctl is-enabled service # Check if enabled
systemctl list-units       # List all units
systemctl list-units --failed # Failed units
```

### Service Dependencies
```bash
# Dependency analysis
systemctl list-dependencies service  # Show dependencies
systemctl show service     # Show all properties
systemctl cat service      # Show service file
systemctl edit service     # Edit service override
```

### Legacy Service Management (SysV)
```bash
# SysV init (older systems)
service service_name start    # Start service
service service_name status   # Check status
chkconfig --list              # List services
chkconfig service_name on     # Enable service
```

---

## Performance Analysis

### CPU Analysis
```bash
# CPU usage
top -bn1 | grep "Cpu(s)"   # Current CPU usage
sar -u 1 10                # CPU utilization over time
mpstat -P ALL 1 5          # Per-CPU statistics
pidstat -u 1 5             # Per-process CPU usage

# CPU information
cat /proc/cpuinfo          # CPU details
lscpu                      # CPU architecture info
nproc                      # Number of processors
```

### Memory Analysis
```bash
# Memory usage
free -m                    # Memory in MB
cat /proc/meminfo          # Detailed memory info
vmstat 1 5                 # Virtual memory stats
sar -r 1 5                 # Memory utilization
ps aux --sort=-%mem | head # Top memory consumers

# Memory debugging
valgrind --leak-check=full program  # Memory leak detection
pmap -x PID                # Detailed process memory map
```

### I/O Analysis
```bash
# Disk I/O
iostat -x 1 5              # Extended I/O statistics
iotop -o                   # I/O by process
sar -d 1 5                 # Disk activity
pidstat -d 1 5             # Per-process disk I/O

# File system I/O
lsof | grep filename       # Processes using file
fuser -v filename          # Verbose file usage
```

---

## Troubleshooting Commands

### System Debugging
```bash
# System calls and debugging
strace command             # Trace system calls
strace -p PID              # Attach to running process
ltrace command             # Trace library calls
gdb program                # GNU debugger

# Process debugging
lsof -p PID                # Files opened by process
cat /proc/PID/status       # Process status
cat /proc/PID/cmdline      # Command line
cat /proc/PID/environ      # Environment variables
```

### Hardware Information
```bash
# Hardware details
lshw                       # List hardware
lshw -short                # Brief hardware list
lspci                      # PCI devices
lsusb                      # USB devices
lsblk                      # Block devices
dmidecode                  # DMI/SMBIOS information
```

### Kernel and Boot
```bash
# Kernel information
uname -a                   # Kernel version and info
cat /proc/version          # Kernel version
dmesg                      # Kernel messages
dmesg | tail               # Recent kernel messages
lsmod                      # Loaded kernel modules
modinfo module_name        # Module information
```

---

## Text Processing

### Basic Text Commands
```bash
# File content
cat file                   # Display file content
less file                  # Page through file
head -n 10 file           # First 10 lines
tail -n 10 file           # Last 10 lines
wc -l file                # Count lines
wc -w file                # Count words
```

### Text Searching and Filtering
```bash
# grep patterns
grep "pattern" file        # Search for pattern
grep -i "pattern" file     # Case insensitive
grep -v "pattern" file     # Invert match
grep -n "pattern" file     # Show line numbers
grep -r "pattern" /path    # Recursive search
grep -E "regex" file       # Extended regex

# Advanced text processing
awk '{print $1}' file      # Print first column
awk '/pattern/ {print $2}' file  # Print second column of matching lines
sed 's/old/new/g' file     # Replace all occurrences
sed -n '10,20p' file       # Print lines 10-20
cut -d: -f1 /etc/passwd    # Extract first field
sort file                  # Sort lines
sort -n file               # Numeric sort
uniq file                  # Remove duplicates
```

### Text Manipulation
```bash
# String operations
tr 'a-z' 'A-Z' < file      # Convert to uppercase
tr -d '\n' < file          # Remove newlines
paste file1 file2          # Merge files side by side
join file1 file2           # Join files on common field
diff file1 file2           # Compare files
comm file1 file2           # Compare sorted files
```

---

## System Information

### System Details
```bash
# Basic system info
hostname                   # System hostname
whoami                     # Current username
id                         # User and group IDs
date                       # Current date and time
uptime                     # System uptime and load
uname -a                   # System information

# Distribution info
cat /etc/os-release        # OS release information
lsb_release -a             # LSB release info
cat /etc/issue             # Issue file
```

### User and Group Information
```bash
# User information
who                        # Logged in users
w                          # Detailed user activity
last                       # Login history
lastlog                    # Last login times
finger username            # User information

# Group information
groups                     # Current user groups
groups username            # User's groups
cat /etc/passwd            # User accounts
cat /etc/group             # Group information
```

### Environment Information
```bash
# Environment variables
env                        # All environment variables
echo $PATH                 # PATH variable
echo $HOME                 # Home directory
printenv                   # Print environment
set                        # Shell variables and functions

# Locale and time
locale                     # Locale settings
timedatectl                # Time and date settings
cal                        # Calendar
```

---

## Quick Reference Commands

### Emergency Commands
```bash
# System recovery
mount -o remount,rw /      # Remount root as read-write
fsck /dev/sda1             # Check filesystem
init 1                     # Single user mode
systemctl rescue           # Rescue mode

# Process management
killall -9 process_name    # Force kill all instances
pkill -f pattern           # Kill by pattern
kill -STOP PID             # Suspend process
kill -CONT PID             # Resume process
```

### One-liners for Common Tasks
```bash
# Find large files
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -hr

# Check disk usage by directory
du -h --max-depth=1 / 2>/dev/null | sort -hr

# Find processes using most CPU
ps aux --sort=-%cpu | head -10

# Find processes using most memory
ps aux --sort=-%mem | head -10

# Check network connections
netstat -tuln | grep LISTEN

# Monitor log files
tail -f /var/log/syslog | grep -i error

# Check service status
systemctl --failed

# Find recently modified files
find /var/log -type f -mtime -1 -exec ls -la {} \;
```

### Interview-Ready Commands
```bash
# System health check
uptime && free -h && df -h && ps aux --sort=-%cpu | head -5

# Network connectivity test
ping -c 3 8.8.8.8 && nslookup google.com

# Service status check
systemctl status sshd nginx mysql

# Log error check
grep -i error /var/log/syslog | tail -10

# Disk I/O check
iostat -x 1 3

# Memory usage by process
ps aux --sort=-%mem | head -10 | awk '{print $2, $4, $11}'
