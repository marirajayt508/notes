# System Logs Analysis Guide - Amazon Lab126 Support Engineer

## Table of Contents
1. [Log Types and Locations](#log-types-and-locations)
2. [Log Formats and Structure](#log-formats-and-structure)
3. [Log Analysis Tools](#log-analysis-tools)
4. [Common Log Patterns](#common-log-patterns)
5. [Real-time Log Monitoring](#real-time-log-monitoring)
6. [Log Rotation and Management](#log-rotation-and-management)
7. [Centralized Logging](#centralized-logging)
8. [Troubleshooting with Logs](#troubleshooting-with-logs)
9. [Python Log Processing](#python-log-processing)
10. [Interview Scenarios](#interview-scenarios)

---

## Log Types and Locations

### System Log Hierarchy
```
/var/log/
├── syslog              # General system messages
├── messages            # System messages (RHEL/CentOS)
├── auth.log            # Authentication logs
├── secure              # Security/auth logs (RHEL/CentOS)
├── kern.log            # Kernel messages
├── dmesg               # Boot messages
├── cron.log            # Cron job logs
├── mail.log            # Mail server logs
├── daemon.log          # System daemon logs
├── user.log            # User-level messages
├── debug               # Debug messages
├── boot.log            # Boot process logs
├── lastlog             # Last login records
├── wtmp                # Login/logout history
├── btmp                # Failed login attempts
└── journal/            # systemd journal files
```

### Application-Specific Logs
```bash
# Web servers
/var/log/apache2/       # Apache logs
/var/log/nginx/         # Nginx logs
/var/log/httpd/         # Apache (RHEL/CentOS)

# Databases
/var/log/mysql/         # MySQL logs
/var/log/postgresql/    # PostgreSQL logs
/var/log/mongodb/       # MongoDB logs

# System services
/var/log/ssh/           # SSH logs
/var/log/dhcp/          # DHCP server logs
/var/log/dns/           # DNS server logs

# Application logs
/var/log/application/   # Custom application logs
/opt/app/logs/          # Application-specific location
```

### Device-Specific Logs (Amazon Devices)
```bash
# Android-based devices (Fire tablets, Fire TV)
/data/logs/             # Application logs
/system/logs/           # System logs
/cache/logs/            # Temporary logs

# Kernel logs
/proc/kmsg              # Kernel message buffer
/dev/log                # System log socket

# Custom device logs
/var/log/device/        # Device-specific logs
/tmp/logs/              # Temporary device logs
```

---

## Log Formats and Structure

### Syslog Format (RFC 3164)
```
<Priority>Timestamp Hostname Tag: Message

Example:
<34>Oct 11 22:14:15 mymachine su: 'su root' failed for lonvick on /dev/pts/8
```

### Syslog Components
```bash
# Priority = Facility * 8 + Severity
# Facilities: 0=kernel, 1=user, 2=mail, 3=daemon, 4=security, etc.
# Severities: 0=emergency, 1=alert, 2=critical, 3=error, 4=warning, 5=notice, 6=info, 7=debug

# Common syslog patterns
Oct 11 22:14:15 hostname process[PID]: message
2023-10-11T22:14:15.123456+00:00 hostname process[PID]: message
```

### Apache Log Formats
```bash
# Common Log Format (CLF)
127.0.0.1 - - [10/Oct/2023:13:55:36 -0700] "GET /index.html HTTP/1.0" 200 2326

# Combined Log Format
127.0.0.1 - - [10/Oct/2023:13:55:36 -0700] "GET /index.html HTTP/1.0" 200 2326 "http://www.example.com/start.html" "Mozilla/4.08"

# Custom format fields
%h - Remote hostname/IP
%l - Remote logname
%u - Remote user
%t - Time of request
%r - First line of request
%s - Status code
%b - Size of response
%{Referer}i - Referer header
%{User-Agent}i - User agent
```

### JSON Log Format
```json
{
  "timestamp": "2023-10-11T22:14:15.123Z",
  "level": "ERROR",
  "service": "api-gateway",
  "message": "Database connection failed",
  "error": {
    "code": "DB_CONN_TIMEOUT",
    "details": "Connection timeout after 30s"
  },
  "request_id": "req-12345",
  "user_id": "user-67890"
}
```

### systemd Journal Format
```bash
# Journal fields
MESSAGE=The actual log message
PRIORITY=Syslog priority value
_PID=Process ID
_UID=User ID
_GID=Group ID
_COMM=Command name
_EXE=Executable path
_SYSTEMD_UNIT=Systemd unit name
_HOSTNAME=Hostname
_TRANSPORT=Transport method
```

---

## Log Analysis Tools

### Basic Command Line Tools
```bash
# View logs
cat /var/log/syslog              # Display entire log
less /var/log/syslog             # Page through log
tail -f /var/log/syslog          # Follow log in real-time
tail -n 100 /var/log/syslog      # Last 100 lines
head -n 50 /var/log/syslog       # First 50 lines

# Search logs
grep "ERROR" /var/log/syslog     # Find errors
grep -i "failed" /var/log/auth.log  # Case insensitive search
grep -v "INFO" /var/log/app.log  # Exclude INFO messages
grep -A 5 -B 5 "ERROR" /var/log/syslog  # Show context lines

# Multiple file search
grep -r "pattern" /var/log/      # Recursive search
zgrep "pattern" /var/log/*.gz    # Search compressed logs
```

### Advanced Text Processing
```bash
# awk for log analysis
awk '/ERROR/ {print $1, $2, $3, $NF}' /var/log/syslog  # Extract specific fields
awk '{print $4}' /var/log/apache2/access.log | sort | uniq -c  # Count by field

# sed for log processing
sed -n '/ERROR/,/INFO/p' /var/log/app.log  # Print between patterns
sed 's/ERROR/CRITICAL/g' /var/log/app.log  # Replace text

# cut for field extraction
cut -d' ' -f1-3 /var/log/syslog   # Extract first 3 fields
cut -d'"' -f2 /var/log/apache2/access.log  # Extract quoted field

# sort and uniq for statistics
sort /var/log/access.log | uniq -c | sort -nr  # Count and sort occurrences
```

### systemd Journal Tools
```bash
# Basic journalctl usage
journalctl                       # All journal entries
journalctl -f                    # Follow journal
journalctl -r                    # Reverse order (newest first)
journalctl -n 50                 # Last 50 entries
journalctl --since "2023-10-11 10:00:00"  # Since specific time
journalctl --until "1 hour ago"  # Until specific time

# Service-specific logs
journalctl -u nginx              # Nginx service logs
journalctl -u ssh                # SSH service logs
journalctl _SYSTEMD_UNIT=nginx.service  # Alternative syntax

# Filter by priority
journalctl -p err                # Error and above
journalctl -p warning..err       # Warning to error range
journalctl -p 0..3               # Numeric priority range

# Filter by process
journalctl _PID=1234             # Specific process ID
journalctl _COMM=sshd            # Specific command
journalctl _UID=1000             # Specific user ID

# Output formats
journalctl -o json               # JSON format
journalctl -o json-pretty        # Pretty JSON
journalctl -o cat                # Message only
journalctl -o short-iso          # ISO timestamp format
```

### Log Analysis with Python
```python
import re
from datetime import datetime
from collections import Counter, defaultdict

def parse_syslog_line(line):
    """Parse a syslog line into components"""
    pattern = r'(\w{3}\s+\d{1,2}\s+\d{2}:\d{2}:\d{2})\s+(\S+)\s+(\S+?):\s+(.*)'
    match = re.match(pattern, line)
    if match:
        return {
            'timestamp': match.group(1),
            'hostname': match.group(2),
            'process': match.group(3),
            'message': match.group(4)
        }
    return None

def analyze_log_file(filename):
    """Analyze log file for patterns and statistics"""
    stats = {
        'total_lines': 0,
        'error_count': 0,
        'warning_count': 0,
        'processes': Counter(),
        'hourly_distribution': defaultdict(int),
        'error_messages': []
    }
    
    with open(filename, 'r') as f:
        for line in f:
            stats['total_lines'] += 1
            
            # Count error levels
            if 'ERROR' in line.upper():
                stats['error_count'] += 1
                stats['error_messages'].append(line.strip())
            elif 'WARNING' in line.upper():
                stats['warning_count'] += 1
            
            # Parse structured logs
            parsed = parse_syslog_line(line)
            if parsed:
                stats['processes'][parsed['process']] += 1
                
                # Extract hour for distribution
                try:
                    time_part = parsed['timestamp'].split()[-1]
                    hour = int(time_part.split(':')[0])
                    stats['hourly_distribution'][hour] += 1
                except (ValueError, IndexError):
                    pass
    
    return stats
```

---

## Common Log Patterns

### Error Patterns to Watch For
```bash
# Authentication failures
grep "Failed password" /var/log/auth.log
grep "authentication failure" /var/log/auth.log
grep "Invalid user" /var/log/auth.log

# System errors
grep -i "error\|fail\|critical\|fatal" /var/log/syslog
grep "segfault\|kernel panic\|oops" /var/log/kern.log
grep "out of memory\|oom" /var/log/syslog

# Network issues
grep "connection refused\|timeout\|unreachable" /var/log/syslog
grep "DNS resolution failed" /var/log/syslog

# Disk issues
grep "I/O error\|disk full\|no space" /var/log/syslog
grep "filesystem.*read-only" /var/log/syslog

# Service failures
grep "failed to start\|service.*failed" /var/log/syslog
systemctl --failed
```

### Performance Indicators in Logs
```bash
# High load indicators
grep "load average" /var/log/syslog
grep "blocked for more than" /var/log/kern.log

# Memory pressure
grep "Out of memory\|oom-killer" /var/log/syslog
grep "Memory cgroup out of memory" /var/log/syslog

# Disk I/O issues
grep "task.*blocked for more than.*seconds" /var/log/kern.log
grep "INFO: task.*blocked" /var/log/kern.log
```

### Security-Related Patterns
```bash
# Suspicious login attempts
grep "Failed password.*root" /var/log/auth.log
awk '/Failed password/ {print $1, $2, $3, $9, $11}' /var/log/auth.log | sort | uniq -c

# Privilege escalation
grep "sudo.*COMMAND" /var/log/auth.log
grep "su.*session opened" /var/log/auth.log

# Network security
grep "DENY\|DROP\|REJECT" /var/log/syslog
grep "port scan\|intrusion" /var/log/syslog
```

---

## Real-time Log Monitoring

### tail Command Variations
```bash
# Basic real-time monitoring
tail -f /var/log/syslog           # Follow single log
tail -f /var/log/syslog /var/log/auth.log  # Multiple logs

# Advanced tail options
tail -F /var/log/syslog           # Follow with retry (handles rotation)
tail -n 0 -f /var/log/syslog      # Start from end of file
tail -c +1 -f /var/log/syslog     # Follow from beginning

# Filtered real-time monitoring
tail -f /var/log/syslog | grep ERROR
tail -f /var/log/apache2/access.log | grep "404\|500"
```

### multitail for Multiple Logs
```bash
# Install multitail (if not available)
# Ubuntu/Debian: apt-get install multitail
# RHEL/CentOS: yum install multitail

# Multiple log windows
multitail /var/log/syslog /var/log/auth.log
multitail -i /var/log/syslog -i /var/log/auth.log

# Filtered monitoring
multitail -e "ERROR" /var/log/syslog -e "Failed" /var/log/auth.log
```

### Real-time Log Analysis Script
```python
#!/usr/bin/env python3
import subprocess
import re
import time
from datetime import datetime

class LogMonitor:
    def __init__(self, log_files, alert_patterns=None):
        self.log_files = log_files if isinstance(log_files, list) else [log_files]
        self.alert_patterns = alert_patterns or [
            r'ERROR',
            r'CRITICAL',
            r'FATAL',
            r'Failed password',
            r'segfault',
            r'out of memory'
        ]
        self.alert_count = 0
    
    def monitor_logs(self):
        """Monitor multiple log files simultaneously"""
        processes = []
        
        # Start tail processes for each log file
        for log_file in self.log_files:
            try:
                process = subprocess.Popen(
                    ['tail', '-F', log_file],
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True,
                    bufsize=1,
                    universal_newlines=True
                )
                processes.append((process, log_file))
            except Exception as e:
                print(f"Error monitoring {log_file}: {e}")
        
        try:
            while True:
                for process, log_file in processes:
                    line = process.stdout.readline()
                    if line:
                        self.process_log_line(line.strip(), log_file)
                time.sleep(0.1)
        except KeyboardInterrupt:
            print("\nStopping log monitoring...")
        finally:
            for process, _ in processes:
                process.terminate()
    
    def process_log_line(self, line, source_file):
        """Process each log line for alerts"""
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        for pattern in self.alert_patterns:
            if re.search(pattern, line, re.IGNORECASE):
                self.alert_count += 1
                severity = self.determine_severity(pattern)
                
                alert_msg = f"[{timestamp}] ALERT #{self.alert_count} ({severity})"
                alert_msg += f"\nSource: {source_file}"
                alert_msg += f"\nPattern: {pattern}"
                alert_msg += f"\nLine: {line}"
                alert_msg += "-" * 80
                
                print(alert_msg)
                
                # Log to alert file
                with open('alerts.log', 'a') as f:
                    f.write(f"{alert_msg}\n")
                
                break
    
    def determine_severity(self, pattern):
        """Determine alert severity based on pattern"""
        high_severity = ['CRITICAL', 'FATAL', 'segfault', 'out of memory']
        medium_severity = ['ERROR', 'Failed password']
        
        pattern_upper = pattern.upper()
        if any(p.upper() in pattern_upper for p in high_severity):
            return 'HIGH'
        elif any(p.upper() in pattern_upper for p in medium_severity):
            return 'MEDIUM'
        else:
            return 'LOW'

# Usage
if __name__ == "__main__":
    monitor = LogMonitor([
        '/var/log/syslog',
        '/var/log/auth.log',
        '/var/log/kern.log'
    ])
    monitor.monitor_logs()
```

---

## Log Rotation and Management

### logrotate Configuration
```bash
# Main configuration file
/etc/logrotate.conf

# Service-specific configurations
/etc/logrotate.d/

# Example logrotate configuration
/var/log/myapp/*.log {
    daily                    # Rotate daily
    missingok               # Don't error if log missing
    rotate 52               # Keep 52 old logs
    compress                # Compress old logs
    delaycompress           # Compress on next rotation
    notifempty              # Don't rotate empty logs
    create 644 root root    # Create new log with permissions
    postrotate              # Commands after rotation
        systemctl reload myapp
    endscript
}
```

### Manual Log Rotation
```bash
# Force log rotation
logrotate -f /etc/logrotate.conf

# Test logrotate configuration
logrotate -d /etc/logrotate.conf

# Rotate specific service
logrotate -f /etc/logrotate.d/nginx

# Check logrotate status
cat /var/lib/logrotate/status
```

### Log Cleanup Scripts
```bash
#!/bin/bash
# cleanup_logs.sh - Clean old log files

LOG_DIR="/var/log"
DAYS_OLD=30

# Find and remove old log files
find $LOG_DIR -name "*.log" -type f -mtime +$DAYS_OLD -delete
find $LOG_DIR -name "*.log.gz" -type f -mtime +$DAYS_OLD -delete

# Clean up rotated logs
find $LOG_DIR -name "*.log.[0-9]*" -type f -mtime +$DAYS_OLD -delete

# Clean up journal logs older than 30 days
journalctl --vacuum-time=30d

echo "Log cleanup completed: $(date)"
```

---

## Centralized Logging

### rsyslog Configuration
```bash
# /etc/rsyslog.conf

# Send logs to remote server
*.* @@logserver.example.com:514

# Receive logs from remote clients
$ModLoad imudp
$UDPServerRun 514
$UDPServerAddress 0.0.0.0

# Template for log format
$template RemoteFormat,"%TIMESTAMP% %HOSTNAME% %syslogtag%%msg%\n"
*.* ?RemoteFormat
```

### syslog-ng Configuration
```bash
# /etc/syslog-ng/syslog-ng.conf

# Source definition
source s_network {
    udp(ip(0.0.0.0) port(514));
    tcp(ip(0.0.0.0) port(514));
};

# Destination definition
destination d_remote {
    file("/var/log/remote/$HOST/$YEAR-$MONTH-$DAY.log"
         create_dirs(yes)
         dir_perm(0755)
         perm(0644));
};

# Log path
log {
    source(s_network);
    destination(d_remote);
};
```

### ELK Stack Basics (Elasticsearch, Logstash, Kibana)
```yaml
# logstash.conf
input {
  file {
    path => "/var/log/syslog"
    start_position => "beginning"
  }
}

filter {
  grok {
    match => { "message" => "%{SYSLOGTIMESTAMP:timestamp} %{IPORHOST:host} %{DATA:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:message}" }
  }
  
  date {
    match => [ "timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "syslog-%{+YYYY.MM.dd}"
  }
}
```

---

## Troubleshooting with Logs

### Common Troubleshooting Scenarios

#### 1. Service Won't Start
```bash
# Check service status
systemctl status service_name

# Check service logs
journalctl -u service_name -f

# Check for configuration errors
journalctl -u service_name --since "10 minutes ago"

# Check system logs for related errors
grep service_name /var/log/syslog
```

#### 2. High CPU Usage
```bash
# Find processes with high CPU
ps aux --sort=-%cpu | head -10

# Check for CPU-related messages
grep -i "cpu\|load" /var/log/syslog

# Monitor system load
tail -f /var/log/syslog | grep "load average"
```

#### 3. Memory Issues
```bash
# Check for OOM killer messages
grep -i "killed process\|out of memory" /var/log/syslog

# Check memory-related errors
dmesg | grep -i memory

# Monitor memory usage in logs
grep -i "memory\|oom" /var/log/kern.log
```

#### 4. Network Connectivity Issues
```bash
# Check network-related errors
grep -i "network\|connection\|timeout" /var/log/syslog

# Check DNS resolution issues
grep -i "dns\|resolution" /var/log/syslog

# Check firewall logs
grep -i "deny\|drop\|reject" /var/log/syslog
```

#### 5. Disk Space Issues
```bash
# Check for disk space errors
grep -i "no space\|disk full" /var/log/syslog

# Check I/O errors
grep -i "i/o error\|input/output error" /var/log/syslog

# Check filesystem errors
grep -i "filesystem.*error" /var/log/syslog
```

### Log-Based Debugging Workflow
```bash
# 1. Identify the problem timeframe
journalctl --since "2023-10-11 14:00:00" --until "2023-10-11 15:00:00"

# 2. Filter by service or component
journalctl -u nginx --since "1 hour ago"

# 3. Look for error patterns
grep -i "error\|fail\|critical" /var/log/syslog

# 4. Correlate across multiple logs
multitail /var/log/syslog /var/log/auth.log /var/log/kern.log

# 5. Extract relevant information
awk '/ERROR/ {print $1, $2, $3, $NF}' /var/log/application.log

# 6. Generate timeline of events
grep "2023-10-11 14:" /var/log/syslog | sort
```

---

## Python Log Processing

### Advanced Log Parser
```python
import re
import json
from datetime import datetime
from collections import defaultdict, Counter

class LogParser:
    def __init__(self):
        self.patterns = {
            'syslog': r'(\w{3}\s+\d{1,2}\s+\d{2}:\d{2}:\d{2})\s+(\S+)\s+(\S+?)(?:\[(\d+)\])?: (.*)',
            'apache_common': r'(\S+) \S+ \S+ \[(.*?)\] "(\S+) (\S+) (\S+)" (\d{3}) (\d+)',
            'apache_combined': r'(\S+) \S+ \S+ \[(.*?)\] "(\S+) (\S+) (\S+)" (\d{3}) (\d+) "(.*?)" "(.*?)"',
            'nginx': r'(\S+) - - \[(.*?)\] "(\w+) (.*?) HTTP/.*?" (\d+) (\d+) "(.*?)" "(.*?)"',
            'json': r'^\{.*\}$'
        }
    
    def detect_format(self, line):
        """Detect log format based on line content"""
        for format_name, pattern in self.patterns.items():
            if re.match(pattern, line):
                return format_name
        return 'unknown'
    
    def parse_line(self, line, format_type=None):
        """Parse a log line based on its format"""
        if not format_type:
            format_type = self.detect_format(line)
        
        if format_type == 'json':
            try:
                return json.loads(line)
            except json.JSONDecodeError:
                return None
        
        pattern = self.patterns.get(format_type)
        if pattern:
            match = re.match(pattern, line)
            if match:
                return self.format_match(match, format_type)
        
        return {'raw': line, 'format': format_type}
    
    def format_match(self, match, format_type):
        """Format regex match based on log type"""
        groups = match.groups()
        
        if format_type == 'syslog':
            return {
                'timestamp': groups[0],
                'hostname': groups[1],
                'process': groups[2],
                'pid': groups[3],
                'message': groups[4],
                'format': format_type
            }
        elif format_type in ['apache_common', 'apache_combined']:
            result = {
                'ip': groups[0],
                'timestamp': groups[1],
                'method': groups[2],
                'url': groups[3],
                'protocol': groups[4],
                'status': int(groups[5]),
                'size': int(groups[6]) if groups[6] != '-' else 0,
                'format': format_type
            }
            if format_type == 'apache_combined' and len(groups) > 7:
                result['referer'] = groups[7]
                result['user_agent'] = groups[8]
            return result
        
        return {'groups': groups, 'format': format_type}

class LogAnalyzer:
    def __init__(self):
        self.parser = LogParser()
        self.stats = defaultdict(int)
        self.errors = []
        self.warnings = []
        self.timeline = []
    
    def analyze_file(self, filename, max_lines=None):
        """Analyze a log file and generate statistics"""
        line_count = 0
        
        with open(filename, 'r', encoding='utf-8', errors='ignore') as f:
            for line in f:
                if max_lines and line_count >= max_lines:
                    break
                
                line = line.strip()
                if not line:
                    continue
                
                line_count += 1
                parsed = self.parser.parse_line(line)
                
                if parsed:
                    self.process_parsed_line(parsed, line_count)
        
        return self.generate_report()
    
    def process_parsed_line(self, parsed, line_number):
        """Process a parsed log line for analysis"""
        self.stats['total_lines'] += 1
        self.stats[f"format_{parsed.get('format', 'unknown')}"] += 1
        
        # Check for errors and warnings
        message = parsed.get('message', parsed.get('raw', ''))
        if isinstance(message, str):
            message_upper = message.upper()
            if any(keyword in message_upper for keyword in ['ERROR', 'CRITICAL', 'FATAL']):
                self.stats['errors'] += 1
                self.errors.append({
                    'line_number': line_number,
                    'content': message,
                    'parsed': parsed
                })
            elif any(keyword in message_upper for keyword in ['WARNING', 'WARN']):
                self.stats['warnings'] += 1
                self.warnings.append({
                    'line_number': line_number,
                    'content': message,
                    'parsed': parsed
                })
        
        # Track HTTP status codes for web logs
        if 'status' in parsed:
            status = parsed['status']
            self.stats[f'http_{status}'] += 1
            if status >= 400:
                self.stats['http_errors'] += 1
        
        # Track IP addresses
        if 'ip' in parsed:
            self.stats['unique_ips'] = len(set(self.stats.get('ip_list', [])))
            if 'ip_list' not in self.stats:
                self.stats['ip_list'] = []
            self.stats['ip_list'].append(parsed['ip'])
    
    def generate_report(self):
        """Generate analysis report"""
        report = {
            'summary': dict(self.stats),
            'top_errors': self.errors[-10:],  # Last 10 errors
            'top_warnings': self.warnings[-10:],  # Last 10 warnings
            'analysis_timestamp': datetime.now().isoformat()
        }
        
        # Clean up large lists from stats
        if 'ip_list' in report['summary']:
            ip_counter = Counter(report['summary']['ip_list'])
            report['top_ips'] = ip_counter.most_common(10)
            del report['summary']['ip_list']
        
        return report

# Usage example
if __name__ == "__main__":
    analyzer = LogAnalyzer()
    report = analyzer.analyze_file('/var/log/syslog', max_lines=1000)
    
    print(json.dumps(report, indent=2, default=str))
```

---

## Interview Scenarios

### Scenario 1: Service Performance Issues
**Question**: "A user reports that the web application is slow. How would you use logs to investigate?"

**Answer Approach**:
```bash
# 1. Check web server access logs for response times
tail -f /var/log/nginx/access.log | awk '{print $NF}' | sort -n

# 2. Look for error patterns
grep -E "5[0-9]{2}|4[0-9]{2}" /var/log/nginx/access.log

# 3. Check application logs for errors
grep -i "error\|exception\|timeout" /var/log/application/app.log

# 4. Correlate with system resource usage
grep "load average\|memory" /var/log/syslog

# 5. Check database logs if applicable
tail -f /var/log/mysql/error.log
```

### Scenario 2: Security Incident Investigation
**Question**: "You suspect unauthorized access attempts. How would you investigate using logs?"

**Answer Approach**:
```bash
# 1. Check authentication logs
grep "Failed password" /var/log/auth.log | tail -20

# 2. Look for successful logins from unusual IPs
awk '/Accepted password/ {print $1, $2, $3, $9, $11}' /var/log/auth.log

# 3. Check for privilege escalation
grep "sudo\|su" /var/log/auth.log | grep -v "session closed"

# 4. Analyze connection patterns
awk '/sshd.*Accepted/ {print $11}' /var/log/auth.log
