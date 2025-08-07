# Amazon Lab126 Support Engineer - Interview Question Bank

## Table of Contents
1. [Technical Questions - Operating Systems](#technical-questions---operating-systems)
2. [Technical Questions - Python/Scripting](#technical-questions---pythonscripting)
3. [Technical Questions - Unix/Linux Commands](#technical-questions---unixlinux-commands)
4. [Technical Questions - Networking](#technical-questions---networking)
5. [Technical Questions - System Troubleshooting](#technical-questions---system-troubleshooting)
6. [Technical Questions - AWS/CloudWatch](#technical-questions---awscloudwatch)
7. [Device-Specific Questions](#device-specific-questions)
8. [Scenario-Based Questions](#scenario-based-questions)
9. [Behavioral Questions](#behavioral-questions)
10. [System Design Questions](#system-design-questions)

---

## Technical Questions - Operating Systems

### Basic OS Concepts
**Q1:** Explain the difference between a process and a thread.
**Expected Answer:** 
- Process: Independent execution unit with its own memory space, file descriptors, and resources
- Thread: Lightweight execution unit within a process, shares memory space and resources
- Threads have lower overhead for creation and context switching
- Inter-thread communication is faster than inter-process communication

**Q2:** What happens when a system runs out of memory?
**Expected Answer:**
- OOM (Out of Memory) killer activates
- Kernel selects processes to terminate based on OOM score
- System may become unresponsive or crash
- Swap space is used if available
- Prevention: monitoring, proper resource allocation, memory limits

**Q3:** Explain the boot process of a Linux system.
**Expected Answer:**
1. BIOS/UEFI performs POST and loads bootloader
2. Bootloader (GRUB) loads kernel and initramfs
3. Kernel initializes hardware and mounts root filesystem
4. Init system (systemd/SysV) starts
5. Services and daemons are started
6. User login becomes available

**Q4:** What is virtual memory and how does it work?
**Expected Answer:**
- Abstraction that gives each process its own address space
- Uses paging to map virtual addresses to physical memory
- Allows running programs larger than physical RAM
- Provides memory protection between processes
- Enables swapping to disk when RAM is full

**Q5:** Describe different process states in Linux.
**Expected Answer:**
- Running (R): Currently executing or ready to run
- Sleeping (S): Waiting for an event (interruptible)
- Uninterruptible Sleep (D): Waiting for I/O operations
- Zombie (Z): Process finished but parent hasn't read exit status
- Stopped (T): Process suspended by signal

### Advanced OS Concepts
**Q6:** How would you debug a process that's consuming 100% CPU?
**Expected Answer:**
```bash
# Identify the process
top -bn1 | head -20
ps aux --sort=-%cpu | head -10

# Analyze the process
strace -p PID              # System call tracing
perf top -p PID            # Performance profiling
gdb -p PID                 # Attach debugger
lsof -p PID                # Check open files

# Check for infinite loops, inefficient algorithms, or resource contention
```

**Q7:** Explain how file permissions work in Unix/Linux.
**Expected Answer:**
- Three permission types: read (r/4), write (w/2), execute (x/1)
- Three user categories: owner, group, others
- Octal notation: 755 = rwxr-xr-x
- Special permissions: setuid (4000), setgid (2000), sticky bit (1000)
- Directory permissions: execute needed to access contents

**Q8:** What is an inode and what information does it contain?
**Expected Answer:**
- Data structure storing file metadata
- Contains: file size, permissions, timestamps, owner/group, link count
- Points to data blocks containing file content
- Doesn't contain filename (stored in directory entry)
- Each file has unique inode number

---

## Technical Questions - Python/Scripting

### Basic Python for System Administration
**Q9:** Write a Python script to monitor disk usage and alert if usage exceeds 80%.
**Expected Answer:**
```python
import shutil
import smtplib
from email.mime.text import MIMEText

def check_disk_usage(path="/", threshold=80):
    total, used, free = shutil.disk_usage(path)
    usage_percent = (used / total) * 100
    
    if usage_percent > threshold:
        send_alert(f"Disk usage at {usage_percent:.1f}% on {path}")
    
    return usage_percent

def send_alert(message):
    # Email alert implementation
    print(f"ALERT: {message}")

if __name__ == "__main__":
    usage = check_disk_usage()
    print(f"Current disk usage: {usage:.1f}%")
```

**Q10:** How would you parse log files to find error patterns using Python?
**Expected Answer:**
```python
import re
from collections import Counter
from datetime import datetime

def analyze_log_file(filename):
    error_patterns = Counter()
    error_times = []
    
    with open(filename, 'r') as f:
        for line in f:
            if 'ERROR' in line.upper():
                # Extract timestamp
                timestamp_match = re.search(r'\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}', line)
                if timestamp_match:
                    error_times.append(timestamp_match.group())
                
                # Count error types
                error_type = extract_error_type(line)
                error_patterns[error_type] += 1
    
    return error_patterns, error_times

def extract_error_type(line):
    # Extract meaningful error pattern
    if 'connection' in line.lower():
        return 'Connection Error'
    elif 'timeout' in line.lower():
        return 'Timeout Error'
    else:
        return 'General Error'
```

**Q11:** Write a script to monitor system resources (CPU, memory, disk).
**Expected Answer:**
```python
import psutil
import time
import json
from datetime import datetime

class SystemMonitor:
    def __init__(self):
        self.thresholds = {
            'cpu': 80,
            'memory': 85,
            'disk': 90
        }
    
    def get_system_stats(self):
        return {
            'timestamp': datetime.now().isoformat(),
            'cpu_percent': psutil.cpu_percent(interval=1),
            'memory_percent': psutil.virtual_memory().percent,
            'disk_percent': psutil.disk_usage('/').percent,
            'load_average': psutil.getloadavg()
        }
    
    def check_thresholds(self, stats):
        alerts = []
        if stats['cpu_percent'] > self.thresholds['cpu']:
            alerts.append(f"High CPU: {stats['cpu_percent']:.1f}%")
        if stats['memory_percent'] > self.thresholds['memory']:
            alerts.append(f"High Memory: {stats['memory_percent']:.1f}%")
        if stats['disk_percent'] > self.thresholds['disk']:
            alerts.append(f"High Disk: {stats['disk_percent']:.1f}%")
        return alerts

monitor = SystemMonitor()
stats = monitor.get_system_stats()
alerts = monitor.check_thresholds(stats)
```

### Advanced Python Questions
**Q12:** How would you implement a log rotation script in Python?
**Expected Answer:**
```python
import os
import gzip
import shutil
from datetime import datetime

def rotate_log(log_file, max_size_mb=100, keep_files=5):
    if not os.path.exists(log_file):
        return
    
    file_size_mb = os.path.getsize(log_file) / (1024 * 1024)
    
    if file_size_mb > max_size_mb:
        # Rotate existing files
        for i in range(keep_files - 1, 0, -1):
            old_file = f"{log_file}.{i}.gz"
            new_file = f"{log_file}.{i + 1}.gz"
            if os.path.exists(old_file):
                if i == keep_files - 1:
                    os.remove(old_file)
                else:
                    shutil.move(old_file, new_file)
        
        # Compress current log
        with open(log_file, 'rb') as f_in:
            with gzip.open(f"{log_file}.1.gz", 'wb') as f_out:
                shutil.copyfileobj(f_in, f_out)
        
        # Create new empty log
        open(log_file, 'w').close()
```

**Q13:** Explain how you would handle exceptions in a system monitoring script.
**Expected Answer:**
```python
import logging
import traceback
from functools import wraps

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('monitor.log'),
        logging.StreamHandler()
    ]
)

def error_handler(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except Exception as e:
            logging.error(f"Error in {func.__name__}: {str(e)}")
            logging.error(f"Traceback: {traceback.format_exc()}")
            return None
    return wrapper

@error_handler
def monitor_system():
    # System monitoring code
    pass
```

---

## Technical Questions - Unix/Linux Commands

### Basic Commands
**Q14:** How would you find all files larger than 100MB in the /var directory?
**Expected Answer:**
```bash
find /var -type f -size +100M -exec ls -lh {} \; 2>/dev/null
# or
find /var -type f -size +100M -print0 | xargs -0 ls -lh
```

**Q15:** Explain the difference between hard links and soft links.
**Expected Answer:**
- Hard link: Direct reference to inode, same file system only, survives if original deleted
- Soft link: Pointer to filename, can cross filesystems, breaks if original deleted
- Commands: `ln file hardlink` vs `ln -s file softlink`

**Q16:** How would you monitor real-time log files for specific patterns?
**Expected Answer:**
```bash
# Basic monitoring
tail -f /var/log/syslog | grep ERROR

# Multiple files
multitail /var/log/syslog /var/log/auth.log

# With alerting
tail -f /var/log/syslog | while read line; do
    if echo "$line" | grep -q "CRITICAL"; then
        echo "ALERT: $line" | mail admin@company.com
    fi
done
```

### Advanced Commands
**Q17:** How would you find and kill all processes belonging to a specific user?
**Expected Answer:**
```bash
# Find processes
ps aux | grep username
pgrep -u username

# Kill processes
pkill -u username
killall -u username

# More controlled approach
ps -u username -o pid= | xargs kill -TERM
```

**Q18:** Explain how to use awk to analyze log files.
**Expected Answer:**
```bash
# Count occurrences by field
awk '{print $1}' access.log | sort | uniq -c | sort -nr

# Extract specific time periods
awk '/2023-10-11 14:/ {print $0}' /var/log/syslog

# Calculate statistics
awk '{sum+=$10; count++} END {print "Average:", sum/count}' data.txt

# Complex pattern matching
awk '/ERROR/ && /database/ {print $1, $2, $NF}' application.log
```

**Q19:** How would you check system performance using command-line tools?
**Expected Answer:**
```bash
# CPU usage
top -bn1 | head -20
sar -u 1 5
vmstat 1 5

# Memory usage
free -h
cat /proc/meminfo
ps aux --sort=-%mem | head -10

# Disk I/O
iostat -x 1 5
iotop -o

# Network
iftop -i eth0
netstat -tuln
ss -tulpn
```

---

## Technical Questions - Networking

### Basic Networking
**Q20:** Explain the OSI model and how you use it for troubleshooting.
**Expected Answer:**
- 7 layers: Physical, Data Link, Network, Transport, Session, Presentation, Application
- Bottom-up troubleshooting approach
- Start with physical (cables, power) and work up to application
- Each layer has specific tools and tests

**Q21:** How would you troubleshoot a "can't reach website" issue?
**Expected Answer:**
```bash
# Step-by-step approach
1. ping 8.8.8.8              # Test internet connectivity
2. nslookup website.com      # Test DNS resolution
3. ping website.com          # Test connectivity to site
4. traceroute website.com    # Check network path
5. telnet website.com 80     # Test port connectivity
6. curl -I website.com       # Test HTTP response
```

**Q22:** What's the difference between TCP and UDP?
**Expected Answer:**
- TCP: Connection-oriented, reliable, ordered delivery, flow control, error checking
- UDP: Connectionless, unreliable, no ordering, lower overhead
- TCP used for: HTTP, HTTPS, SSH, FTP
- UDP used for: DNS, DHCP, streaming media, gaming

### Advanced Networking
**Q23:** How would you diagnose high network latency?
**Expected Answer:**
```bash
# Measure latency
ping -c 100 target_host
mtr --report-cycles 50 target_host

# Analyze network path
traceroute -n target_host
pathping target_host  # Windows

# Check for packet loss
ping -f -c 1000 target_host  # Flood ping (careful!)

# Monitor network utilization
iftop -i eth0
nethogs
```

**Q24:** Explain how DNS resolution works and how to troubleshoot DNS issues.
**Expected Answer:**
1. Client checks local cache
2. Queries local DNS resolver
3. Resolver queries root servers
4. Root refers to TLD servers
5. TLD refers to authoritative servers
6. Answer returned and cached

Troubleshooting:
```bash
nslookup domain.com
dig domain.com
dig @8.8.8.8 domain.com
host domain.com
```

---

## Technical Questions - System Troubleshooting

### Performance Issues
**Q25:** A server is running slowly. Walk me through your troubleshooting process.
**Expected Answer:**
1. Check system load: `uptime`, `top`
2. Identify resource bottleneck:
   - CPU: `top`, `sar -u`
   - Memory: `free -h`, `vmstat`
   - Disk: `iostat`, `iotop`
   - Network: `iftop`, `nethogs`
3. Identify problematic processes
4. Check logs for errors
5. Analyze recent changes
6. Implement solution and monitor

**Q26:** How would you troubleshoot a process that's stuck in "D" state?
**Expected Answer:**
- "D" state = uninterruptible sleep (usually I/O wait)
- Check what the process is waiting for:
```bash
cat /proc/PID/stack
cat /proc/PID/wchan
lsof -p PID
strace -p PID
```
- Common causes: disk I/O issues, NFS problems, hardware failures
- May require system reboot if persistent

**Q27:** A system won't boot. How do you troubleshoot?
**Expected Answer:**
1. Check hardware (power, connections, POST)
2. Boot from rescue media
3. Check filesystem: `fsck /dev/sda1`
4. Mount root filesystem and check logs
5. Check boot configuration (GRUB)
6. Verify kernel and initramfs
7. Check for disk space issues
8. Review recent changes

### Application Issues
**Q28:** An application keeps crashing. How do you investigate?
**Expected Answer:**
1. Check application logs
2. Check system logs: `journalctl -u service_name`
3. Check resource usage: memory leaks, CPU spikes
4. Use debugging tools:
```bash
strace -p PID
gdb -p PID
valgrind --leak-check=full ./application
```
5. Check dependencies and libraries
6. Review recent code changes
7. Check configuration files

---

## Technical Questions - AWS/CloudWatch

### CloudWatch Basics
**Q29:** How would you set up monitoring for an Amazon device fleet?
**Expected Answer:**
1. Define key metrics: CPU, memory, network, application-specific
2. Create custom namespaces for device types
3. Use dimensions for filtering (DeviceId, DeviceType, Location)
4. Set up alarms with appropriate thresholds
5. Create dashboards for visualization
6. Implement log aggregation
7. Set up SNS notifications

**Q30:** Explain how you would optimize CloudWatch costs.
**Expected Answer:**
- Set appropriate log retention periods
- Use metric filters instead of custom metrics where possible
- Reduce metric collection frequency for non-critical metrics
- Delete unused log groups and metrics
- Use composite alarms to reduce alarm count
- Implement log sampling for high-volume applications

**Q31:** How would you create a CloudWatch alarm for device connectivity?
**Expected Answer:**
```python
import boto3

cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_alarm(
    AlarmName='DeviceConnectivityAlarm',
    ComparisonOperator='LessThanThreshold',
    EvaluationPeriods=2,
    MetricName='ConnectedDevices',
    Namespace='Amazon/Devices',
    Period=300,
    Statistic='Average',
    Threshold=100.0,
    ActionsEnabled=True,
    AlarmActions=[
        'arn:aws:sns:us-east-1:123456789012:device-alerts'
    ],
    AlarmDescription='Alert when device connectivity drops',
    Dimensions=[
        {
            'Name': 'DeviceType',
            'Value': 'FireTV'
        }
    ]
)
```

---

## Device-Specific Questions

### Fire TV Questions
**Q32:** A customer reports their Fire TV is buffering during Netflix streaming. How do you troubleshoot?
**Expected Answer:**
1. Check internet speed requirements (5+ Mbps for HD)
2. Test network connectivity and latency
3. Check for network congestion
4. Verify streaming service status
5. Check device temperature and performance
6. Clear app cache and data
7. Test with different streaming services
8. Check WiFi signal strength and interference

**Q33:** Fire TV won't connect to WiFi. What are the troubleshooting steps?
**Expected Answer:**
1. Verify WiFi password and network name
2. Check network band compatibility (2.4GHz vs 5GHz)
3. Restart router and Fire TV
4. Check router security settings (WPA2/WPA3)
5. Test with mobile hotspot
6. Reset network settings on device
7. Check for MAC address filtering
8. Update Fire TV software

### Echo Device Questions
**Q34:** An Echo device isn't responding to voice commands. How do you diagnose?
**Expected Answer:**
1. Check microphone mute status
2. Verify internet connectivity
3. Test network latency (should be <200ms)
4. Check for background noise interference
5. Verify account linking in Alexa app
6. Test wake word sensitivity
7. Check device placement and acoustics
8. Restart Echo device

**Q35:** How would you troubleshoot Echo smart home integration issues?
**Expected Answer:**
1. Check device discovery in Alexa app
2. Verify smart home device connectivity
3. Test individual device control
4. Check skill linking and permissions
5. Review device groupings and scenes
6. Test voice commands syntax
7. Check for skill updates
8. Re-link skills if necessary

---

## Scenario-Based Questions

### Scenario 1: Production Outage
**Q36:** You receive an alert that all Fire TV devices in a region can't stream content. Walk me through your response.
**Expected Answer:**
1. **Immediate Assessment** (0-5 minutes):
   - Confirm scope and impact
   - Check service status dashboards
   - Verify monitoring systems
   - Notify stakeholders

2. **Initial Investigation** (5-15 minutes):
   - Check CDN and streaming service status
   - Review recent deployments
   - Check network connectivity
   - Analyze error patterns in logs

3. **Diagnosis** (15-30 minutes):
   - Correlate logs across services
   - Check infrastructure components
   - Test from different locations
   - Identify root cause

4. **Resolution** (30+ minutes):
   - Implement fix or rollback
   - Monitor recovery
   - Communicate status updates
   - Document incident

### Scenario 2: Performance Degradation
**Q37:** Multiple customers report slow performance on their Kindle devices. How do you investigate?
**Expected Answer:**
1. **Data Collection**:
   - Gather device metrics (CPU, memory, network)
   - Check service response times
   - Analyze user reports for patterns
   - Review recent changes

2. **Analysis**:
   - Identify affected device models/regions
   - Check backend service performance
   - Analyze network latency and throughput
   - Review content delivery metrics

3. **Root Cause Investigation**:
   - Check for resource bottlenecks
   - Analyze database performance
   - Review caching effectiveness
   - Check for memory leaks

4. **Resolution**:
   - Optimize slow queries
   - Scale resources if needed
   - Implement caching improvements
   - Deploy performance fixes

### Scenario 3: Security Incident
**Q38:** You notice unusual network traffic from Echo devices. How do you respond?
**Expected Answer:**
1. **Immediate Response**:
   - Isolate affected devices if possible
   - Preserve evidence and logs
   - Notify security team
   - Document timeline

2. **Investigation**:
   - Analyze network traffic patterns
   - Check for malware or compromise
   - Review authentication logs
   - Identify attack vectors

3. **Containment**:
   - Block malicious traffic
   - Update security rules
   - Patch vulnerabilities
   - Reset compromised credentials

4. **Recovery**:
   - Restore clean device images
   - Verify system integrity
   - Monitor for reoccurrence
   - Update security procedures

---

## Behavioral Questions

### Amazon Leadership Principles
**Q39:** Tell me about a time you had to dive deep to solve a complex technical problem. (Dive Deep)
**Expected Answer Structure:**
- **Situation**: Describe the complex problem
- **Task**: Your responsibility in solving it
- **Action**: Specific steps you took to investigate deeply
- **Result**: Outcome and what you learned

**Q40:** Describe a time when you had to work with incomplete information to solve a problem. (Bias for Action)
**Expected Answer Structure:**
- **Situation**: Problem with limited information
- **Task**: Need to make progress despite uncertainty
- **Action**: Steps taken to move forward while gathering info
- **Result**: Outcome and lessons learned

**Q41:** Tell me about a time you made a mistake. How did you handle it? (Earn Trust)
**Expected Answer Structure:**
- **Situation**: Describe the mistake honestly
- **Task**: Your responsibility to address it
- **Action**: How you took ownership and fixed it
- **Result**: What you learned and how you prevented recurrence

### Technical Collaboration
**Q42:** Describe a time you had to explain a complex technical issue to a non-technical stakeholder.
**Expected Answer Structure:**
- Use analogies and simple language
- Focus on business impact
- Provide clear next steps
- Confirm understanding

**Q43:** Tell me about a time you disagreed with a technical decision. How did you handle it?
**Expected Answer Structure:**
- Present your concerns professionally
- Provide data to support your position
- Listen to other perspectives
- Work toward consensus or escalate appropriately

---

## System Design Questions

### Monitoring System Design
**Q44:** Design a monitoring system for Amazon's device fleet (Fire TV, Echo, Kindle).
**Expected Answer:**
1. **Requirements**:
   - Monitor millions of devices
   - Real-time alerting
   - Historical data analysis
   - Cost-effective solution

2. **Architecture**:
   - Device agents collect metrics
   - CloudWatch for metrics and logs
   - SNS for notifications
   - Lambda for processing
   - DynamoDB for device metadata

3. **Key Metrics**:
   - Device health (CPU, memory, temperature)
   - Network connectivity and performance
   - Application-specific metrics
   - User experience metrics

4. **Scalability**:
   - Batch metric submissions
   - Regional deployment
   - Auto-scaling components
   - Cost optimization strategies

**Q45:** How would you design a log aggregation system for device troubleshooting?
**Expected Answer:**
1. **Collection**:
   - Device agents with configurable log levels
   - Secure transmission to cloud
   - Compression and batching

2. **Storage**:
   - CloudWatch Logs for real-time analysis
   - S3 for long-term storage
   - Partitioning by device type and date

3. **Processing**:
   - Lambda for real-time processing
   - Kinesis for streaming analytics
   - EMR for batch processing

4. **Analysis**:
   - CloudWatch Logs Insights for queries
   - Elasticsearch for complex searches
   - Machine learning for anomaly detection

### Troubleshooting Workflow Design
**Q46:** Design an automated troubleshooting system for common device issues.
**Expected Answer:**
1. **Issue Detection**:
   - Automated monitoring and alerting
   - Customer report integration
   - Pattern recognition

2. **Diagnosis**:
   - Decision tree for common issues
   - Automated data collection
   - Remote diagnostic commands

3. **Resolution**:
   - Automated fixes for simple issues
   - Escalation to human support
   - Self-service options for customers

4. **Learning**:
   - Feedback loop for improvement
   - Knowledge base updates
   - Pattern analysis for new issues

---

## Preparation Tips

### Technical Preparation
1. **Hands-on Practice**: Set up a Linux environment and practice commands
2. **Scripting**: Write Python scripts for system administration tasks
3. **AWS Experience**: Get familiar with CloudWatch, EC2, and basic AWS services
4. **Device Knowledge**: Research Amazon device specifications and common issues

### Interview Preparation
1. **STAR Method**: Structure behavioral answers using Situation, Task, Action, Result
2. **Leadership Principles**: Prepare examples for each Amazon Leadership Principle
3. **Technical Depth**: Be ready to dive deep into any topic you mention
4. **Questions to Ask**: Prepare thoughtful questions about the role and team

### Common Mistakes to Avoid
1. **Not asking clarifying questions** before solving problems
2. **Jumping to solutions** without proper diagnosis
3. **Not considering scalability** in system design questions
4. **Forgetting to mention monitoring and alerting** in solutions
5. **Not explaining your thought process** clearly

### Final Interview Tips
- **Think out loud** during technical problems
- **Start with simple solutions** then add complexity
- **Consider edge cases** and error handling
- **Mention trade-offs** in your solutions
- **Be honest** about what you don't know
- **Show enthusiasm** for learning and problem-solving

This question bank covers the breadth and depth expected for an Amazon Lab126 Support Engineer position. Practice these questions, understand the underlying concepts, and be prepared to dive deep into any technical area.
