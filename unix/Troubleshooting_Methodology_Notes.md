# Troubleshooting Methodology Notes - Amazon Lab126 Support Engineer

## Table of Contents
1. [Systematic Troubleshooting Approach](#systematic-troubleshooting-approach)
2. [Problem Identification](#problem-identification)
3. [Data Collection](#data-collection)
4. [Root Cause Analysis](#root-cause-analysis)
5. [Common Issue Categories](#common-issue-categories)
6. [Device-Specific Troubleshooting](#device-specific-troubleshooting)
7. [Performance Issues](#performance-issues)
8. [Network Connectivity Problems](#network-connectivity-problems)
9. [System Resource Issues](#system-resource-issues)
10. [Application and Service Issues](#application-and-service-issues)
11. [Escalation Guidelines](#escalation-guidelines)
12. [Documentation and Follow-up](#documentation-and-follow-up)

---

## Systematic Troubleshooting Approach

### The ITIL Incident Management Process
```
1. Incident Detection/Reporting
2. Incident Logging
3. Incident Categorization
4. Incident Prioritization
5. Initial Diagnosis
6. Incident Escalation (if needed)
7. Investigation and Diagnosis
8. Resolution and Recovery
9. Incident Closure
```

### The 5-Step Troubleshooting Method
```
Step 1: IDENTIFY the problem
Step 2: ESTABLISH a theory of probable cause
Step 3: TEST the theory to determine cause
Step 4: ESTABLISH a plan of action and implement solution
Step 5: VERIFY full system functionality and document
```

### OSI Layer Troubleshooting (Bottom-Up Approach)
```
Layer 1 (Physical):     Check cables, power, hardware
Layer 2 (Data Link):    Check MAC addresses, switches, VLANs
Layer 3 (Network):      Check IP configuration, routing
Layer 4 (Transport):    Check ports, firewalls, TCP/UDP
Layer 5-7 (Application): Check services, applications, protocols
```

---

## Problem Identification

### Gathering Initial Information
```bash
# Essential Questions to Ask
1. What exactly is the problem?
2. When did it start?
3. Has it happened before?
4. What changed recently?
5. Who is affected?
6. What is the business impact?
7. Are there any error messages?
8. What troubleshooting has already been done?
```

### Problem Classification
```bash
# Severity Levels
Critical (P1):    Complete service outage, security breach
High (P2):        Major functionality impaired, many users affected
Medium (P3):      Minor functionality impaired, few users affected
Low (P4):         Cosmetic issues, single user affected

# Categories
Hardware:         Physical device failures
Software:         Application bugs, OS issues
Network:          Connectivity, performance, configuration
Security:         Unauthorized access, data breaches
Performance:      Slow response times, resource exhaustion
```

### Creating a Problem Statement
```
Template: "Users are experiencing [specific symptom] when [specific action] 
on [specific system/device] since [timeframe], affecting [scope of impact]."

Example: "Fire TV users are experiencing buffering during Netflix streaming 
on all devices in the living room since yesterday evening, affecting 
video quality for the entire household."
```

---

## Data Collection

### System Information Gathering
```bash
# Basic System Info
uname -a                    # System information
cat /etc/os-release         # OS version
uptime                      # System uptime and load
who                         # Current users
last                        # Login history
df -h                       # Disk usage
free -h                     # Memory usage
ps aux --sort=-%cpu | head  # Top CPU processes

# Hardware Information
lscpu                       # CPU information
lshw -short                 # Hardware summary
lspci                       # PCI devices
lsusb                       # USB devices
dmesg | tail -20            # Recent kernel messages
```

### Log Collection Strategy
```bash
# System Logs
journalctl -xe              # Recent systemd logs
tail -f /var/log/syslog     # System messages
grep -i error /var/log/syslog | tail -20  # Recent errors
dmesg | grep -i error       # Kernel errors

# Application Logs
tail -f /var/log/application.log
grep -i "$(date '+%Y-%m-%d')" /var/log/app.log  # Today's logs
find /var/log -name "*.log" -mtime -1  # Recently modified logs

# Network Logs
grep -i "network\|connection" /var/log/syslog
journalctl -u NetworkManager -f
```

### Performance Data Collection
```bash
# CPU and Memory
top -bn1 | head -20         # Current system state
vmstat 1 5                  # Virtual memory stats
iostat -x 1 5               # I/O statistics
sar -u 1 5                  # CPU utilization

# Network Performance
ping -c 10 8.8.8.8          # Network connectivity
traceroute google.com       # Network path
netstat -tuln               # Network connections
ss -tulpn                   # Socket statistics

# Disk Performance
iotop -o                    # I/O by process
lsof +D /path               # Files open in directory
```

---

## Root Cause Analysis

### The 5 Whys Technique
```
Example: Fire TV won't connect to WiFi

Why 1: Why won't the Fire TV connect to WiFi?
Answer: The device shows "Authentication failed"

Why 2: Why is authentication failing?
Answer: The password appears to be incorrect

Why 3: Why is the password incorrect?
Answer: The WiFi password was recently changed

Why 4: Why wasn't the Fire TV password updated?
Answer: No one knew the Fire TV needed manual password update

Why 5: Why wasn't there a process to update all devices?
Answer: No documented procedure for WiFi password changes

Root Cause: Lack of device management procedure for network changes
```

### Fishbone Diagram Categories
```
People:           User error, training issues, staffing
Process:          Procedures, workflows, documentation
Technology:       Hardware, software, network, tools
Environment:      Physical conditions, power, temperature
Materials:        Cables, components, consumables
Methods:          Troubleshooting approaches, standards
```

### Timeline Analysis
```bash
# Create incident timeline
echo "Incident Timeline Analysis" > timeline.txt
echo "=========================" >> timeline.txt
echo "$(date): Problem reported" >> timeline.txt

# Correlate with system events
grep "$(date '+%Y-%m-%d')" /var/log/syslog | grep -E "(error|fail|critical)"

# Check for recent changes
find /etc -name "*.conf" -mtime -7  # Config changes in last week
rpm -qa --last | head -10           # Recent package installations (RHEL/CentOS)
dpkg.log | grep "$(date '+%Y-%m-%d')"  # Recent package changes (Debian/Ubuntu)
```

---

## Common Issue Categories

### 1. Boot and Startup Issues
```bash
# Troubleshooting Steps
1. Check boot logs: journalctl -b
2. Verify filesystem: fsck /dev/sda1
3. Check disk space: df -h
4. Review service failures: systemctl --failed
5. Check hardware: dmesg | grep -i error

# Common Causes
- Filesystem corruption
- Disk space exhaustion
- Hardware failures
- Configuration errors
- Service dependencies
```

### 2. Performance Degradation
```bash
# Investigation Steps
1. Identify resource bottleneck:
   - CPU: top, htop, sar -u
   - Memory: free, vmstat, ps aux --sort=-%mem
   - Disk: iostat, iotop, df -h
   - Network: iftop, nethogs, ping

2. Check for resource-intensive processes:
   ps aux --sort=-%cpu | head -10
   ps aux --sort=-%mem | head -10

3. Analyze system load:
   uptime
   cat /proc/loadavg
   sar -q 1 5

# Common Solutions
- Kill resource-intensive processes
- Increase system resources
- Optimize applications
- Clean up disk space
- Tune system parameters
```

### 3. Network Connectivity Issues
```bash
# Layer-by-Layer Diagnosis
1. Physical Layer:
   - Check cable connections
   - Verify link lights
   - Test with different cables

2. Data Link Layer:
   - Check interface status: ip link show
   - Verify MAC addresses: ip neigh show
   - Check switch configuration

3. Network Layer:
   - Verify IP configuration: ip addr show
   - Test gateway connectivity: ping gateway_ip
   - Check routing: ip route show

4. Transport Layer:
   - Test port connectivity: nc -zv host port
   - Check firewall rules: iptables -L
   - Verify service status: systemctl status service

5. Application Layer:
   - Test application functionality
   - Check application logs
   - Verify configuration files
```

---

## Device-Specific Troubleshooting

### Fire TV Troubleshooting
```bash
# Common Issues and Solutions

1. Won't Connect to WiFi:
   - Check WiFi password
   - Verify network band (2.4GHz vs 5GHz)
   - Check router settings (WPA2/WPA3)
   - Reset network settings on device
   - Restart router and Fire TV

2. Streaming Issues (Buffering):
   - Test internet speed: iperf3 -c server
   - Check for network congestion: iftop
   - Verify streaming service status
   - Clear app cache and data
   - Check for device overheating

3. App Crashes:
   - Check available storage space
   - Clear app cache: Settings > Applications
   - Update app to latest version
   - Restart Fire TV device
   - Check system logs for errors

# Diagnostic Commands (if accessible via ADB)
adb devices                 # List connected devices
adb shell                   # Access device shell
adb logcat                  # View device logs
adb shell dumpsys meminfo   # Memory information
```

### Echo Device Troubleshooting
```bash
# Common Issues and Solutions

1. Not Responding to Voice Commands:
   - Check microphone mute status
   - Verify internet connectivity: ping alexa.amazon.com
   - Test voice recognition in Alexa app
   - Check for background noise interference
   - Restart Echo device

2. Music Playback Issues:
   - Verify music service subscription
   - Check account linking in Alexa app
   - Test with different music services
   - Check network bandwidth: > 512 Kbps required
   - Restart music service skill

3. Smart Home Control Issues:
   - Verify device discovery: "Alexa, discover devices"
   - Check smart home device connectivity
   - Review device groupings in Alexa app
   - Test individual device control
   - Re-link smart home skills if needed

# Network Requirements Check
ping -c 5 alexa.amazon.com  # Should be < 200ms latency
nc -zv alexa.amazon.com 443 # HTTPS connectivity
nc -zv alexa.amazon.com 4070 # Voice service port
```

### Kindle Troubleshooting
```bash
# Common Issues and Solutions

1. Won't Download Books:
   - Check WiFi connectivity
   - Verify Amazon account status
   - Check available storage space
   - Sync device: Settings > Sync
   - Restart Kindle device

2. Slow Performance:
   - Check available storage (keep 10% free)
   - Close unused applications
   - Restart device
   - Check for software updates
   - Reset to factory settings (last resort)

3. Battery Issues:
   - Check charging cable and adapter
   - Verify charging port cleanliness
   - Monitor battery usage patterns
   - Adjust screen brightness
   - Disable WiFi when not needed

# Network Requirements
- Minimum bandwidth: 256 Kbps
- Ports: 80 (HTTP), 443 (HTTPS)
- DNS resolution: kindle.amazon.com
```

---

## Performance Issues

### CPU Performance Problems
```bash
# Diagnosis Steps
1. Identify high CPU processes:
   top -bn1 | head -20
   ps aux --sort=-%cpu | head -10

2. Check CPU utilization over time:
   sar -u 1 10
   vmstat 1 10

3. Analyze load average:
   uptime
   cat /proc/loadavg

4. Check for CPU-bound processes:
   ps -eo pid,ppid,cmd,pcpu --sort=-pcpu | head -10

# Common Solutions
- Kill unnecessary processes: kill -TERM PID
- Reduce process priority: renice +10 PID
- Optimize application code
- Add more CPU cores (if virtual)
- Schedule intensive tasks during off-peak hours
```

### Memory Performance Problems
```bash
# Diagnosis Steps
1. Check memory usage:
   free -h
   cat /proc/meminfo

2. Identify memory-hungry processes:
   ps aux --sort=-%mem | head -10
   pmap -x PID  # Detailed memory map

3. Check for memory leaks:
   valgrind --leak-check=full ./program
   ps -o pid,vsz,rss,comm -p PID  # Monitor over time

4. Analyze swap usage:
   swapon -s
   vmstat 1 5

# Common Solutions
- Kill memory-intensive processes
- Increase swap space
- Add more RAM
- Fix memory leaks in applications
- Tune kernel memory parameters
```

### Disk Performance Problems
```bash
# Diagnosis Steps
1. Check disk usage:
   df -h
   du -sh /* | sort -rh | head -10

2. Monitor I/O performance:
   iostat -x 1 5
   iotop -o

3. Check for disk errors:
   dmesg | grep -i "i/o error"
   smartctl -a /dev/sda

4. Analyze disk queue length:
   sar -d 1 5

# Common Solutions
- Clean up unnecessary files: find /tmp -mtime +7 -delete
- Move files to different disks
- Optimize database queries
- Add more storage or faster disks
- Implement disk caching
```

---

## Network Connectivity Problems

### No Internet Access
```bash
# Systematic Diagnosis
1. Check physical connectivity:
   ip link show  # Interface status
   ethtool eth0  # Link details

2. Verify IP configuration:
   ip addr show
   ip route show

3. Test local network:
   ping gateway_ip
   ping 192.168.1.1

4. Test DNS resolution:
   nslookup google.com
   dig @8.8.8.8 google.com

5. Test external connectivity:
   ping 8.8.8.8
   traceroute google.com

# Step-by-step Resolution
if ! ping -c 1 gateway_ip; then
    echo "Gateway unreachable - check local network"
elif ! nslookup google.com; then
    echo "DNS issue - check /etc/resolv.conf"
elif ! ping -c 1 8.8.8.8; then
    echo "Internet connectivity issue - check ISP"
else
    echo "Connectivity OK - check application-specific issues"
fi
```

### Slow Network Performance
```bash
# Performance Testing
1. Test bandwidth:
   iperf3 -c iperf.he.net
   speedtest-cli

2. Check for packet loss:
   ping -c 100 8.8.8.8
   mtr --report-cycles 50 google.com

3. Analyze network path:
   traceroute -n google.com
   pathping google.com  # Windows

4. Monitor network utilization:
   iftop -i eth0
   nethogs

# Common Causes and Solutions
- Network congestion: Schedule heavy transfers
- DNS issues: Use faster DNS servers (8.8.8.8, 1.1.1.1)
- Routing problems: Check routing tables
- Hardware issues: Replace network equipment
- QoS settings: Prioritize critical traffic
```

---

## System Resource Issues

### Disk Space Issues
```bash
# Quick Diagnosis
df -h | grep -E "(9[0-9]%|100%)"  # Find full filesystems

# Detailed Analysis
1. Find large files:
   find / -type f -size +100M 2>/dev/null | head -20
   du -ah / | sort -rh | head -20

2. Check log file sizes:
   du -sh /var/log/*
   find /var/log -name "*.log" -size +50M

3. Identify temporary files:
   find /tmp -type f -atime +7
   find /var/tmp -type f -atime +7

# Cleanup Solutions
# Clean package cache
apt-get clean          # Debian/Ubuntu
yum clean all          # RHEL/CentOS

# Rotate logs
logrotate -f /etc/logrotate.conf

# Clean temporary files
find /tmp -type f -atime +7 -delete
find /var/tmp -type f -atime +7 -delete

# Remove old kernels (Ubuntu)
apt-get autoremove --purge
```

### File System Corruption
```bash
# Detection
dmesg | grep -i "filesystem\|corruption\|error"
mount | grep ro  # Check for read-only mounts

# Repair Process
1. Unmount filesystem (if possible):
   umount /dev/sda1

2. Check and repair:
   fsck -y /dev/sda1      # Auto-repair
   e2fsck -f /dev/sda1    # Force check (ext2/3/4)
   xfs_repair /dev/sda1   # XFS repair

3. Remount filesystem:
   mount /dev/sda1 /mnt

# Prevention
- Regular filesystem checks
- Monitor disk health with SMART
- Maintain adequate free space
- Use journaling filesystems
```

---

## Application and Service Issues

### Service Won't Start
```bash
# Diagnosis Steps
1. Check service status:
   systemctl status service_name
   systemctl is-enabled service_name

2. Check service logs:
   journalctl -u service_name -f
   journalctl -u service_name --since "1 hour ago"

3. Check configuration:
   systemctl cat service_name
   nginx -t  # Test nginx config
   apache2ctl configtest  # Test Apache config

4. Check dependencies:
   systemctl list-dependencies service_name
   systemctl show service_name

# Common Solutions
- Fix configuration errors
- Start required dependencies
- Check file permissions
- Verify user/group ownership
- Check available resources (ports, memory)
```

### Application Performance Issues
```bash
# Monitoring Application Performance
1. Check application logs:
   tail -f /var/log/application.log
   grep -i "slow\|timeout\|error" /var/log/app.log

2. Monitor resource usage:
   pidstat -u -r -d 1 5  # CPU, memory, disk I/O
   strace -p PID          # System call tracing

3. Check database performance:
   mysql> SHOW PROCESSLIST;
   postgresql: SELECT * FROM pg_stat_activity;

4. Analyze network connections:
   netstat -tulpn | grep :80
   ss -tulpn | grep :3306

# Performance Tuning
- Optimize database queries
- Implement caching
- Tune application parameters
- Scale horizontally or vertically
- Use connection pooling
```

---

## Escalation Guidelines

### When to Escalate
```bash
# Escalation Triggers
1. Severity Level:
   - P1 issues after 15 minutes
   - P2 issues after 1 hour
   - P3 issues after 4 hours

2. Technical Complexity:
   - Requires specialized knowledge
   - Involves multiple systems
   - Security implications
   - Vendor-specific issues

3. Resource Requirements:
   - Need additional tools/access
   - Require hardware replacement
   - Need management approval
   - Budget implications
```

### Escalation Information Package
```bash
# Information to Provide
1. Problem Summary:
   - Clear problem statement
   - Business impact assessment
   - Affected users/systems
   - Timeline of events

2. Technical Details:
   - Error messages and logs
   - System configuration
   - Troubleshooting steps taken
   - Test results and findings

3. Environmental Information:
   - System specifications
   - Network topology
   - Recent changes
   - Related incidents

# Escalation Template
Subject: [P1/P2/P3] - Brief Problem Description
Problem: Detailed problem description
Impact: Business impact and affected users
Timeline: When problem started and key events
Troubleshooting: Steps already taken
Next Steps: Recommended actions
Contact: Your contact information
```

---

## Documentation and Follow-up

### Incident Documentation Template
```markdown
# Incident Report: [Incident ID]

## Summary
- **Date/Time**: 2023-10-11 14:30 UTC
- **Duration**: 2 hours 15 minutes
- **Severity**: P2 - High
- **Status**: Resolved

## Problem Description
Brief description of the issue and its impact.

## Timeline
- 14:30 - Issue reported by user
- 14:35 - Initial investigation started
- 15:00 - Root cause identified
- 15:30 - Fix implemented
- 16:45 - Full service restored

## Root Cause
Detailed explanation of what caused the issue.

## Resolution
Steps taken to resolve the issue.

## Prevention
Measures to prevent recurrence:
- Process improvements
- Monitoring enhancements
- Training requirements
- System changes

## Lessons Learned
Key takeaways from this incident.
```

### Knowledge Base Entry
```markdown
# KB Article: [Title]

## Problem
Common symptoms and error messages.

## Cause
Technical explanation of the root cause.

## Solution
Step-by-step resolution procedure.

## Prevention
How to avoid this issue in the future.

## Related Articles
Links to related documentation.

## Tags
Keywords for searchability.
```

### Post-Incident Review
```bash
# Review Questions
1. What went well during the incident response?
2. What could have been done better?
3. Were there any delays in detection or response?
4. Was the communication effective?
5. Are there process improvements needed?
6. What monitoring/alerting gaps were identified?
7. What training needs were identified?
8. Are there any system improvements needed?
```

---

## Quick Reference Troubleshooting Commands

### System Health Check
```bash
#!/bin/bash
# Quick system health check script

echo "=== System Health Check ==="
echo "Date: $(date)"
echo

echo "=== System Load ==="
uptime
echo

echo "=== Memory Usage ==="
free -h
echo

echo "=== Disk Usage ==="
df -h | grep -E "(Filesystem|/dev/)"
echo

echo "=== Top CPU Processes ==="
ps aux --sort=-%cpu | head -6
echo

echo "=== Top Memory Processes ==="
ps aux --sort=-%mem | head -6
echo

echo "=== Network Connectivity ==="
ping -c 3 8.8.8.8 | tail -2
echo

echo "=== Recent Errors ==="
journalctl -p err --since "1 hour ago" --no-pager | tail -5
```

### Network Troubleshooting Script
```bash
#!/bin/bash
# Network connectivity troubleshooting

TARGET=${1:-google.com}

echo "=== Network Troubleshooting for $TARGET ==="
echo

echo "=== Interface Status ==="
ip addr show | grep -E "(inet |state )"
echo

echo "=== Default Gateway ==="
ip route show default
echo

echo "=== DNS Resolution ==="
nslookup $TARGET
echo

echo "=== Connectivity Test ==="
ping -c 4 $TARGET
echo

echo "=== Traceroute ==="
traceroute -n $TARGET | head -10
```

This comprehensive troubleshooting guide provides systematic approaches for diagnosing and resolving issues across all major categories you'll encounter as an Amazon Lab126 Support Engineer. Use it as both a learning resource and a quick reference during actual troubleshooting scenarios.
