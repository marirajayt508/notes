# Python System Programming - Amazon Lab126 Support Engineer

## Table of Contents
1. [OS Module Deep Dive](#os-module-deep-dive)
2. [Process Management](#process-management)
3. [File Operations](#file-operations)
4. [System Monitoring](#system-monitoring)
5. [Log Processing](#log-processing)
6. [Network Programming](#network-programming)
7. [Error Handling](#error-handling)
8. [Automation Scripts](#automation-scripts)
9. [Interview Code Examples](#interview-code-examples)

---

## OS Module Deep Dive

### Environment Variables
```python
import os

# Access environment variables
home_dir = os.environ.get('HOME', '/tmp')
path = os.environ['PATH']

# Set environment variables
os.environ['MY_VAR'] = 'value'

# Get all environment variables
for key, value in os.environ.items():
    print(f"{key}={value}")
```

### Process Information
```python
import os
import sys

# Process information
print(f"Process ID: {os.getpid()}")
print(f"Parent Process ID: {os.getppid()}")
print(f"User ID: {os.getuid()}")
print(f"Group ID: {os.getgid()}")
print(f"Current working directory: {os.getcwd()}")

# Change directory
os.chdir('/tmp')

# Command line arguments
print(f"Script name: {sys.argv[0]}")
print(f"Arguments: {sys.argv[1:]}")
```

### File System Operations
```python
import os
import stat

# Directory operations
os.makedirs('/path/to/dir', exist_ok=True)
os.rmdir('/path/to/empty/dir')
os.removedirs('/path/to/nested/dirs')

# File operations
os.rename('old_name', 'new_name')
os.remove('filename')
os.chmod('filename', 0o755)

# Path operations
print(os.path.exists('/path/to/file'))
print(os.path.isfile('/path/to/file'))
print(os.path.isdir('/path/to/dir'))
print(os.path.getsize('/path/to/file'))

# Walk directory tree
for root, dirs, files in os.walk('/path'):
    for file in files:
        filepath = os.path.join(root, file)
        print(filepath)
```

---

## Process Management

### Subprocess Module
```python
import subprocess
import sys

# Run command and capture output
result = subprocess.run(['ls', '-la'], 
                       capture_output=True, 
                       text=True)
print(f"Return code: {result.returncode}")
print(f"Stdout: {result.stdout}")
print(f"Stderr: {result.stderr}")

# Run command with shell
result = subprocess.run('ps aux | grep python', 
                       shell=True, 
                       capture_output=True, 
                       text=True)

# Real-time output
process = subprocess.Popen(['tail', '-f', '/var/log/syslog'],
                          stdout=subprocess.PIPE,
                          stderr=subprocess.PIPE,
                          text=True)

# Read output line by line
for line in process.stdout:
    print(line.strip())
```

### Process Monitoring with psutil
```python
import psutil
import time

# System information
print(f"CPU count: {psutil.cpu_count()}")
print(f"Memory: {psutil.virtual_memory()}")
print(f"Disk usage: {psutil.disk_usage('/')}")

# CPU usage
cpu_percent = psutil.cpu_percent(interval=1)
print(f"CPU usage: {cpu_percent}%")

# Memory usage
memory = psutil.virtual_memory()
print(f"Memory usage: {memory.percent}%")
print(f"Available memory: {memory.available / (1024**3):.2f} GB")

# Process information
for proc in psutil.process_iter(['pid', 'name', 'cpu_percent', 'memory_percent']):
    try:
        if proc.info['cpu_percent'] > 50:
            print(f"High CPU process: {proc.info}")
    except (psutil.NoSuchProcess, psutil.AccessDenied):
        pass

# Disk I/O
disk_io = psutil.disk_io_counters()
print(f"Disk read: {disk_io.read_bytes}")
print(f"Disk write: {disk_io.write_bytes}")

# Network I/O
net_io = psutil.net_io_counters()
print(f"Bytes sent: {net_io.bytes_sent}")
print(f"Bytes received: {net_io.bytes_recv}")
```

### Signal Handling
```python
import signal
import time
import sys

def signal_handler(signum, frame):
    print(f"Received signal {signum}")
    sys.exit(0)

# Register signal handlers
signal.signal(signal.SIGINT, signal_handler)   # Ctrl+C
signal.signal(signal.SIGTERM, signal_handler)  # Termination

# Send signal to process
import os
os.kill(pid, signal.SIGUSR1)

# Ignore signal
signal.signal(signal.SIGPIPE, signal.SIG_IGN)
```

---

## File Operations

### File I/O Best Practices
```python
import os
import json
import csv

# Safe file reading
def read_file_safe(filename):
    try:
        with open(filename, 'r') as f:
            return f.read()
    except FileNotFoundError:
        print(f"File {filename} not found")
        return None
    except PermissionError:
        print(f"Permission denied: {filename}")
        return None

# File writing with backup
def write_file_with_backup(filename, content):
    backup_name = f"{filename}.backup"
    
    # Create backup if file exists
    if os.path.exists(filename):
        os.rename(filename, backup_name)
    
    try:
        with open(filename, 'w') as f:
            f.write(content)
        # Remove backup on success
        if os.path.exists(backup_name):
            os.remove(backup_name)
    except Exception as e:
        # Restore backup on failure
        if os.path.exists(backup_name):
            os.rename(backup_name, filename)
        raise e

# Process large files efficiently
def process_large_file(filename):
    with open(filename, 'r') as f:
        for line_num, line in enumerate(f, 1):
            # Process line by line to save memory
            if 'ERROR' in line:
                print(f"Line {line_num}: {line.strip()}")

# JSON operations
def read_json_config(filename):
    try:
        with open(filename, 'r') as f:
            return json.load(f)
    except json.JSONDecodeError as e:
        print(f"Invalid JSON in {filename}: {e}")
        return None

# CSV operations
def process_csv_file(filename):
    with open(filename, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            print(row)
```

### File Monitoring
```python
import os
import time
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class LogFileHandler(FileSystemEventHandler):
    def on_modified(self, event):
        if not event.is_directory:
            print(f"File modified: {event.src_path}")
            self.process_log_file(event.src_path)
    
    def process_log_file(self, filepath):
        # Process new log entries
        with open(filepath, 'r') as f:
            f.seek(0, 2)  # Go to end of file
            while True:
                line = f.readline()
                if not line:
                    break
                if 'ERROR' in line:
                    print(f"Error found: {line.strip()}")

# Monitor directory for changes
observer = Observer()
observer.schedule(LogFileHandler(), '/var/log', recursive=True)
observer.start()
```

---

## System Monitoring

### System Health Monitor
```python
import psutil
import time
import logging
from datetime import datetime

class SystemMonitor:
    def __init__(self, thresholds=None):
        self.thresholds = thresholds or {
            'cpu': 80,
            'memory': 85,
            'disk': 90
        }
        
        # Setup logging
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler('system_monitor.log'),
                logging.StreamHandler()
            ]
        )
        self.logger = logging.getLogger(__name__)
    
    def check_cpu(self):
        cpu_percent = psutil.cpu_percent(interval=1)
        if cpu_percent > self.thresholds['cpu']:
            self.logger.warning(f"High CPU usage: {cpu_percent}%")
            return False
        return True
    
    def check_memory(self):
        memory = psutil.virtual_memory()
        if memory.percent > self.thresholds['memory']:
            self.logger.warning(f"High memory usage: {memory.percent}%")
            return False
        return True
    
    def check_disk(self):
        disk = psutil.disk_usage('/')
        disk_percent = (disk.used / disk.total) * 100
        if disk_percent > self.thresholds['disk']:
            self.logger.warning(f"High disk usage: {disk_percent:.1f}%")
            return False
        return True
    
    def get_top_processes(self, sort_by='cpu', limit=5):
        processes = []
        for proc in psutil.process_iter(['pid', 'name', 'cpu_percent', 'memory_percent']):
            try:
                processes.append(proc.info)
            except (psutil.NoSuchProcess, psutil.AccessDenied):
                pass
        
        return sorted(processes, 
                     key=lambda x: x[f'{sort_by}_percent'], 
                     reverse=True)[:limit]
    
    def generate_report(self):
        report = {
            'timestamp': datetime.now().isoformat(),
            'cpu_percent': psutil.cpu_percent(),
            'memory': psutil.virtual_memory()._asdict(),
            'disk': psutil.disk_usage('/')._asdict(),
            'top_cpu_processes': self.get_top_processes('cpu'),
            'top_memory_processes': self.get_top_processes('memory')
        }
        return report
    
    def run_continuous_monitoring(self, interval=60):
        while True:
            self.check_cpu()
            self.check_memory()
            self.check_disk()
            time.sleep(interval)

# Usage
monitor = SystemMonitor()
report = monitor.generate_report()
print(json.dumps(report, indent=2))
```

### Service Health Checker
```python
import subprocess
import requests
import socket
from datetime import datetime

class ServiceHealthChecker:
    def __init__(self):
        self.services = [
            'nginx',
            'apache2',
            'mysql',
            'postgresql'
        ]
        self.ports = [80, 443, 22, 3306, 5432]
        self.urls = [
            'http://localhost',
            'https://api.example.com/health'
        ]
    
    def check_service_status(self, service_name):
        try:
            result = subprocess.run(
                ['systemctl', 'is-active', service_name],
                capture_output=True,
                text=True
            )
            return result.stdout.strip() == 'active'
        except Exception as e:
            print(f"Error checking service {service_name}: {e}")
            return False
    
    def check_port(self, host, port, timeout=5):
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(timeout)
            result = sock.connect_ex((host, port))
            sock.close()
            return result == 0
        except Exception:
            return False
    
    def check_url(self, url, timeout=10):
        try:
            response = requests.get(url, timeout=timeout)
            return response.status_code == 200
        except Exception:
            return False
    
    def run_health_check(self):
        results = {
            'timestamp': datetime.now().isoformat(),
            'services': {},
            'ports': {},
            'urls': {}
        }
        
        # Check services
        for service in self.services:
            results['services'][service] = self.check_service_status(service)
        
        # Check ports
        for port in self.ports:
            results['ports'][port] = self.check_port('localhost', port)
        
        # Check URLs
        for url in self.urls:
            results['urls'][url] = self.check_url(url)
        
        return results

# Usage
checker = ServiceHealthChecker()
health_status = checker.run_health_check()
print(json.dumps(health_status, indent=2))
```

---

## Log Processing

### Log Parser and Analyzer
```python
import re
import json
from datetime import datetime
from collections import defaultdict, Counter

class LogAnalyzer:
    def __init__(self):
        self.log_patterns = {
            'apache': r'(\S+) \S+ \S+ \[([\w:/]+\s[+\-]\d{4})\] "(\S+) (\S+) (\S+)" (\d{3}) (\d+)',
            'nginx': r'(\S+) - - \[(.*?)\] "(\w+) (.*?) HTTP/.*?" (\d+) (\d+)',
            'syslog': r'(\w+\s+\d+\s+\d+:\d+:\d+) (\S+) (\S+): (.*)',
            'error': r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) \[(\w+)\] (.*)'
        }
    
    def parse_log_line(self, line, log_type='syslog'):
        pattern = self.log_patterns.get(log_type)
        if pattern:
            match = re.match(pattern, line)
            if match:
                return match.groups()
        return None
    
    def analyze_log_file(self, filename, log_type='syslog'):
        stats = {
            'total_lines': 0,
            'error_count': 0,
            'warning_count': 0,
            'ip_addresses': Counter(),
            'error_messages': Counter(),
            'hourly_distribution': defaultdict(int)
        }
        
        with open(filename, 'r') as f:
            for line in f:
                stats['total_lines'] += 1
                
                # Count error levels
                if 'ERROR' in line.upper():
                    stats['error_count'] += 1
                elif 'WARNING' in line.upper():
                    stats['warning_count'] += 1
                
                # Extract IP addresses
                ip_pattern = r'\b(?:\d{1,3}\.){3}\d{1,3}\b'
                ips = re.findall(ip_pattern, line)
                for ip in ips:
                    stats['ip_addresses'][ip] += 1
                
                # Parse structured logs
                parsed = self.parse_log_line(line, log_type)
                if parsed and log_type == 'apache':
                    timestamp_str = parsed[1]
                    try:
                        timestamp = datetime.strptime(timestamp_str.split()[0], '%d/%b/%Y:%H:%M:%S')
                        hour = timestamp.hour
                        stats['hourly_distribution'][hour] += 1
                    except ValueError:
                        pass
        
        return stats
    
    def find_error_patterns(self, filename):
        error_patterns = []
        with open(filename, 'r') as f:
            for line_num, line in enumerate(f, 1):
                if 'ERROR' in line.upper():
                    error_patterns.append({
                        'line_number': line_num,
                        'content': line.strip(),
                        'timestamp': self.extract_timestamp(line)
                    })
        return error_patterns
    
    def extract_timestamp(self, line):
        # Common timestamp patterns
        patterns = [
            r'\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}',
            r'\d{2}/\w{3}/\d{4}:\d{2}:\d{2}:\d{2}',
            r'\w{3} \d{2} \d{2}:\d{2}:\d{2}'
        ]
        
        for pattern in patterns:
            match = re.search(pattern, line)
            if match:
                return match.group()
        return None
    
    def generate_report(self, stats):
        report = f"""
Log Analysis Report
==================
Total Lines: {stats['total_lines']}
Errors: {stats['error_count']}
Warnings: {stats['warning_count']}

Top IP Addresses:
{chr(10).join([f"  {ip}: {count}" for ip, count in stats['ip_addresses'].most_common(10)])}

Hourly Distribution:
{chr(10).join([f"  {hour:02d}:00 - {count} entries" for hour, count in sorted(stats['hourly_distribution'].items())])}
        """
        return report

# Usage
analyzer = LogAnalyzer()
stats = analyzer.analyze_log_file('/var/log/syslog')
print(analyzer.generate_report(stats))
```

### Real-time Log Monitor
```python
import subprocess
import re
import time
from datetime import datetime

class LogMonitor:
    def __init__(self, log_file, alert_patterns=None):
        self.log_file = log_file
        self.alert_patterns = alert_patterns or [
            r'ERROR',
            r'CRITICAL',
            r'FATAL',
            r'Exception',
            r'failed'
        ]
        self.alerts = []
    
    def tail_log(self):
        process = subprocess.Popen(
            ['tail', '-f', self.log_file],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )
        
        try:
            for line in process.stdout:
                self.process_log_line(line.strip())
        except KeyboardInterrupt:
            process.terminate()
    
    def process_log_line(self, line):
        timestamp = datetime.now().isoformat()
        
        # Check for alert patterns
        for pattern in self.alert_patterns:
            if re.search(pattern, line, re.IGNORECASE):
                alert = {
                    'timestamp': timestamp,
                    'pattern': pattern,
                    'line': line,
                    'severity': self.determine_severity(pattern)
                }
                self.alerts.append(alert)
                self.handle_alert(alert)
    
    def determine_severity(self, pattern):
        severity_map = {
            'CRITICAL': 'high',
            'FATAL': 'high',
            'ERROR': 'medium',
            'Exception': 'medium',
            'failed': 'low'
        }
        return severity_map.get(pattern.upper(), 'low')
    
    def handle_alert(self, alert):
        print(f"ALERT [{alert['severity'].upper()}]: {alert['line']}")
        
        # Send notification for high severity
        if alert['severity'] == 'high':
            self.send_notification(alert)
    
    def send_notification(self, alert):
        # Implement notification logic (email, Slack, etc.)
        print(f"NOTIFICATION SENT: {alert['pattern']} detected")

# Usage
monitor = LogMonitor('/var/log/syslog')
monitor.tail_log()
```

---

## Network Programming

### Network Utilities
```python
import socket
import requests
import subprocess
import json

class NetworkUtils:
    @staticmethod
    def check_port(host, port, timeout=5):
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(timeout)
            result = sock.connect_ex((host, port))
            sock.close()
            return result == 0
        except Exception:
            return False
    
    @staticmethod
    def get_local_ip():
        try:
            # Connect to a remote address to determine local IP
            sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
            sock.connect(("8.8.8.8", 80))
            local_ip = sock.getsockname()[0]
            sock.close()
            return local_ip
        except Exception:
            return "127.0.0.1"
    
    @staticmethod
    def ping_host(host, count=4):
        try:
            result = subprocess.run(
                ['ping', '-c', str(count), host],
                capture_output=True,
                text=True
            )
            return {
                'success': result.returncode == 0,
                'output': result.stdout,
                'error': result.stderr
            }
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    @staticmethod
    def traceroute(host):
        try:
            result = subprocess.run(
                ['traceroute', host],
                capture_output=True,
                text=True,
                timeout=30
            )
            return {
                'success': result.returncode == 0,
                'output': result.stdout,
                'error': result.stderr
            }
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    @staticmethod
    def check_dns(domain):
        try:
            ip = socket.gethostbyname(domain)
            return {'success': True, 'ip': ip}
        except socket.gaierror as e:
            return {'success': False, 'error': str(e)}
    
    @staticmethod
    def check_http_endpoint(url, timeout=10):
        try:
            response = requests.get(url, timeout=timeout)
            return {
                'success': True,
                'status_code': response.status_code,
                'response_time': response.elapsed.total_seconds(),
                'headers': dict(response.headers)
            }
        except Exception as e:
            return {'success': False, 'error': str(e)}

# Usage
net_utils = NetworkUtils()
print(f"Local IP: {net_utils.get_local_ip()}")
print(f"Port 80 open: {net_utils.check_port('google.com', 80)}")
print(f"DNS lookup: {net_utils.check_dns('google.com')}")
```

---

## Error Handling

### Robust Error Handling Patterns
```python
import logging
import traceback
import functools
from contextlib import contextmanager

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('application.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

def retry_on_failure(max_retries=3, delay=1):
    """Decorator to retry function on failure"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        logger.error(f"Function {func.__name__} failed after {max_retries} attempts: {e}")
                        raise
                    logger.warning(f"Attempt {attempt + 1} failed for {func.__name__}: {e}")
                    time.sleep(delay)
            return None
        return wrapper
    return decorator

@contextmanager
def error_handler(operation_name):
    """Context manager for error handling"""
    try:
        logger.info(f"Starting {operation_name}")
        yield
        logger.info(f"Completed {operation_name}")
    except Exception as e:
        logger.error(f"Error in {operation_name}: {e}")
        logger.error(f"Traceback: {traceback.format_exc()}")
        raise

class SystemOperationError(Exception):
    """Custom exception for system operations"""
    pass

def safe_system_operation(operation_func):
    """Wrapper for safe system operations"""
    try:
        result = operation_func()
        return {'success': True, 'result': result}
    except subprocess.CalledProcessError as e:
        error_msg = f"Command failed: {e.cmd}, Return code: {e.returncode}"
        logger.error(error_msg)
        return {'success': False, 'error': error_msg}
    except FileNotFoundError as e:
        error_msg = f"File not found: {e.filename}"
        logger.error(error_msg)
        return {'success': False, 'error': error_msg}
    except PermissionError as e:
        error_msg = f"Permission denied: {e.filename}"
        logger.error(error_msg)
        return {'success': False, 'error': error_msg}
    except Exception as e:
        error_msg = f"Unexpected error: {str(e)}"
        logger.error(error_msg)
        logger.error(f"Traceback: {traceback.format_exc()}")
        return {'success': False, 'error': error_msg}

# Usage examples
@retry_on_failure(max_retries=3, delay=2)
def unreliable_network_call():
    response = requests.get('https://api.example.com/data')
    return response.json()

with error_handler("File processing"):
    process_large_file('/path/to/file')
```

---

## Automation Scripts

### System Maintenance Script
```python
#!/usr/bin/env python3
import os
import subprocess
import logging
import argparse
from datetime import datetime, timedelta

class SystemMaintenance:
    def __init__(self, dry_run=False):
        self.dry_run = dry_run
        self.logger = self.setup_logging()
    
    def setup_logging(self):
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler(f'maintenance_{datetime.now().strftime("%Y%m%d")}.log'),
                logging.StreamHandler()
            ]
        )
        return logging.getLogger(__name__)
    
    def cleanup_old_logs(self, log_dir='/var/log', days_old=30):
        """Remove log files older than specified days"""
        cutoff_date = datetime.now() - timedelta(days=days_old)
        removed_files = []
        
        for root, dirs, files in os.walk(log_dir):
            for file in files:
                if file.endswith('.log') or file.endswith('.log.gz'):
                    filepath = os.path.join(root, file)
                    try:
                        file_time = datetime.fromtimestamp(os.path.getmtime(filepath))
                        if file_time < cutoff_date:
                            if not self.dry_run:
                                os.remove(filepath)
                            removed_files.append(filepath)
                            self.logger.info(f"{'Would remove' if self.dry_run else 'Removed'}: {filepath}")
                    except (OSError, ValueError) as e:
                        self.logger.error(f"Error processing {filepath}: {e}")
        
        return removed_files
    
    def cleanup_temp_files(self, temp_dirs=['/tmp', '/var/tmp']):
        """Clean up temporary files"""
        cleaned_files = []
        
        for temp_dir in temp_dirs:
            if not os.path.exists(temp_dir):
                continue
                
            for root, dirs, files in os.walk(temp_dir):
                for file in files:
                    filepath = os.path.join(root, file)
                    try:
                        # Remove files older than 7 days
                        file_time = datetime.fromtimestamp(os.path.getmtime(filepath))
                        if file_time < datetime.now() - timedelta(days=7):
                            if not self.dry_run:
                                os.remove(filepath)
                            cleaned_files.append(filepath)
                            self.logger.info(f"{'Would remove' if self.dry_run else 'Removed'}: {filepath}")
                    except (OSError, ValueError) as e:
                        self.logger.error(f"Error processing {filepath}: {e}")
        
        return cleaned_files
    
    def rotate_logs(self):
        """Rotate system logs"""
        try:
            if not self.dry_run:
                result = subprocess.run(['logrotate', '/etc/logrotate.conf'], 
                                      capture_output=True, text=True)
                if result.returncode == 0:
                    self.logger.info("Log rotation completed successfully")
                else:
                    self.logger.error(f"Log rotation failed: {result.stderr}")
            else:
                self.logger.info("Would run log rotation")
        except Exception as e:
            self.logger.error(f"Error running log rotation: {e}")
    
    def check_disk_usage(self, threshold=90):
        """Check disk usage and alert if above threshold"""
        try:
            result = subprocess.run(['df', '-h'], capture_output=True, text=True)
            lines = result.stdout.strip().split('\n')[1:]  # Skip header
            
            alerts = []
            for line in lines:
                parts = line.split()
                if len(parts) >= 5:
                    usage_str = parts[4].rstrip('%')
                    if usage_str.isdigit():
                        usage = int(usage_str)
                        if usage > threshold:
                            alerts.append(f"High disk usage: {parts[5]} ({usage}%)")
                            self.logger.warning(f"High disk usage: {parts[5]} ({usage}%)")
            
            return alerts
        except Exception as e:
            self.logger.error(f"Error checking disk usage: {e}")
            return []
    
    def run_maintenance(self):
        """Run all maintenance tasks"""
        self.logger.info("Starting system maintenance")
        
        # Check disk usage first
        disk_alerts = self.check_disk_usage()
        
        # Clean up old logs
        removed_logs = self.cleanup_old_logs()
        
        # Clean up temp files
        cleaned_temps = self.cleanup_temp_files()
        
        # Rotate logs
        self.rotate_logs()
        
        # Generate summary
        summary = {
            'timestamp': datetime.now().isoformat(),
            'disk_alerts': disk_alerts,
            'removed_logs': len(removed_logs),
            'cleaned_temp_files': len(cleaned_temps),
            'dry_run': self.dry_run
        }
        
        self.logger.info(f"Maintenance completed: {summary}")
        return summary

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='System Maintenance Script')
    parser.add_argument('--dry-run', action='store_true', 
                       help='Show what would be done without making changes')
    args = parser.parse_args()
