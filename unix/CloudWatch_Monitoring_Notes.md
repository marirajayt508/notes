# CloudWatch Monitoring Notes - Amazon Lab126 Support Engineer

## Table of Contents
1. [CloudWatch Overview](#cloudwatch-overview)
2. [Core Components](#core-components)
3. [Metrics and Monitoring](#metrics-and-monitoring)
4. [Logs Management](#logs-management)
5. [Alarms and Notifications](#alarms-and-notifications)
6. [Dashboards](#dashboards)
7. [CloudWatch Agent](#cloudwatch-agent)
8. [Device Monitoring](#device-monitoring)
9. [Cost Optimization](#cost-optimization)
10. [Troubleshooting](#troubleshooting)
11. [CLI and API Usage](#cli-and-api-usage)
12. [Interview Scenarios](#interview-scenarios)

---

## CloudWatch Overview

### What is CloudWatch?
Amazon CloudWatch is a monitoring and observability service that provides:
- **Metrics**: Numerical data about your resources and applications
- **Logs**: Centralized log management and analysis
- **Alarms**: Automated responses to metric thresholds
- **Events**: System events and scheduled actions
- **Dashboards**: Visual representation of metrics and logs

### Key Benefits for Device Support
- **Real-time Monitoring**: Track device performance and health
- **Centralized Logging**: Aggregate logs from multiple devices
- **Automated Alerting**: Proactive issue detection
- **Historical Analysis**: Trend analysis and capacity planning
- **Integration**: Works with other AWS services

### CloudWatch Architecture
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Data Sources  │───▶│   CloudWatch    │───▶│   Actions       │
│                 │    │                 │    │                 │
│ • EC2 Instances │    │ • Metrics       │    │ • Alarms        │
│ • Applications  │    │ • Logs          │    │ • Notifications │
│ • Custom Metrics│    │ • Events        │    │ • Auto Scaling  │
│ • AWS Services  │    │ • Dashboards    │    │ • Lambda        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## Core Components

### 1. Metrics
```bash
# Metric Components
- Namespace: Container for metrics (e.g., AWS/EC2, Custom/Application)
- Metric Name: Name of the metric (e.g., CPUUtilization)
- Dimensions: Name/value pairs that identify metric (e.g., InstanceId=i-1234567890abcdef0)
- Timestamp: Time when metric was recorded
- Value: Metric value
- Unit: Unit of measurement (e.g., Percent, Count, Bytes)
```

### 2. Logs
```bash
# Log Components
- Log Groups: Collection of log streams with same retention and access control
- Log Streams: Sequence of log events from same source
- Log Events: Individual log entries with timestamp and message
- Retention Period: How long logs are stored (1 day to never expire)
```

### 3. Alarms
```bash
# Alarm States
- OK: Metric is within threshold
- ALARM: Metric has breached threshold
- INSUFFICIENT_DATA: Not enough data to determine state

# Alarm Components
- Metric: What to monitor
- Threshold: Value that triggers alarm
- Comparison Operator: How to compare (>, <, >=, <=)
- Evaluation Periods: How many periods to evaluate
- Datapoints to Alarm: How many datapoints must breach threshold
```

---

## Metrics and Monitoring

### Default AWS Metrics
```bash
# EC2 Metrics (5-minute intervals by default)
CPUUtilization          # Percentage of CPU usage
DiskReadOps            # Number of read operations
DiskWriteOps           # Number of write operations
DiskReadBytes          # Bytes read from disk
DiskWriteBytes         # Bytes written to disk
NetworkIn              # Bytes received
NetworkOut             # Bytes sent
NetworkPacketsIn       # Packets received
NetworkPacketsOut      # Packets sent

# EBS Metrics
VolumeReadOps          # Read operations
VolumeWriteOps         # Write operations
VolumeReadBytes        # Bytes read
VolumeWriteBytes       # Bytes written
VolumeTotalReadTime    # Total read time
VolumeTotalWriteTime   # Total write time
VolumeIdleTime         # Idle time
VolumeQueueLength      # Queue length
```

### Custom Metrics for Device Monitoring
```python
import boto3
from datetime import datetime

class DeviceMetrics:
    def __init__(self, region='us-east-1'):
        self.cloudwatch = boto3.client('cloudwatch', region_name=region)
        self.namespace = 'Amazon/Devices'
    
    def put_metric(self, metric_name, value, unit='Count', dimensions=None):
        """Send custom metric to CloudWatch"""
        try:
            response = self.cloudwatch.put_metric_data(
                Namespace=self.namespace,
                MetricData=[
                    {
                        'MetricName': metric_name,
                        'Value': value,
                        'Unit': unit,
                        'Timestamp': datetime.utcnow(),
                        'Dimensions': dimensions or []
                    }
                ]
            )
            return response
        except Exception as e:
            print(f"Error sending metric: {e}")
            return None
    
    def put_device_health_metrics(self, device_id, cpu_usage, memory_usage, 
                                 disk_usage, network_latency):
        """Send device health metrics"""
        dimensions = [
            {
                'Name': 'DeviceId',
                'Value': device_id
            }
        ]
        
        metrics = [
            ('CPUUtilization', cpu_usage, 'Percent'),
            ('MemoryUtilization', memory_usage, 'Percent'),
            ('DiskUtilization', disk_usage, 'Percent'),
            ('NetworkLatency', network_latency, 'Milliseconds')
        ]
        
        for metric_name, value, unit in metrics:
            self.put_metric(metric_name, value, unit, dimensions)
    
    def put_application_metrics(self, app_name, response_time, error_count, 
                               request_count):
        """Send application performance metrics"""
        dimensions = [
            {
                'Name': 'ApplicationName',
                'Value': app_name
            }
        ]
        
        metrics = [
            ('ResponseTime', response_time, 'Milliseconds'),
            ('ErrorCount', error_count, 'Count'),
            ('RequestCount', request_count, 'Count')
        ]
        
        for metric_name, value, unit in metrics:
            self.put_metric(metric_name, value, unit, dimensions)

# Usage
device_metrics = DeviceMetrics()
device_metrics.put_device_health_metrics(
    device_id='fire-tv-001',
    cpu_usage=45.2,
    memory_usage=67.8,
    disk_usage=23.1,
    network_latency=15.3
)
```

### Metric Math and Statistics
```python
# Get metric statistics
def get_metric_statistics(metric_name, start_time, end_time, namespace='AWS/EC2'):
    cloudwatch = boto3.client('cloudwatch')
    
    response = cloudwatch.get_metric_statistics(
        Namespace=namespace,
        MetricName=metric_name,
        Dimensions=[
            {
                'Name': 'InstanceId',
                'Value': 'i-1234567890abcdef0'
            }
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,  # 5 minutes
        Statistics=['Average', 'Maximum', 'Minimum']
    )
    
    return response['Datapoints']

# Metric math expressions
def create_metric_math_alarm():
    cloudwatch = boto3.client('cloudwatch')
    
    cloudwatch.put_metric_alarm(
        AlarmName='HighErrorRate',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=2,
        Metrics=[
            {
                'Id': 'm1',
                'MetricStat': {
                    'Metric': {
                        'Namespace': 'AWS/ApplicationELB',
                        'MetricName': 'HTTPCode_Target_4XX_Count',
                        'Dimensions': [
                            {
                                'Name': 'LoadBalancer',
                                'Value': 'app/my-load-balancer/50dc6c495c0c9188'
                            }
                        ]
                    },
                    'Period': 300,
                    'Stat': 'Sum'
                }
            },
            {
                'Id': 'm2',
                'MetricStat': {
                    'Metric': {
                        'Namespace': 'AWS/ApplicationELB',
                        'MetricName': 'RequestCount',
                        'Dimensions': [
                            {
                                'Name': 'LoadBalancer',
                                'Value': 'app/my-load-balancer/50dc6c495c0c9188'
                            }
                        ]
                    },
                    'Period': 300,
                    'Stat': 'Sum'
                }
            },
            {
                'Id': 'e1',
                'Expression': 'm1/m2*100'
            }
        ],
        Threshold=5.0,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:my-topic'
        ]
    )
```

---

## Logs Management

### CloudWatch Logs Structure
```bash
# Hierarchy
Account
└── Region
    └── Log Group (/aws/lambda/my-function)
        └── Log Stream (2023/10/11/[$LATEST]abcd1234)
            └── Log Events (individual log entries)
```

### Log Groups and Streams
```python
import boto3
import json
from datetime import datetime

class CloudWatchLogs:
    def __init__(self, region='us-east-1'):
        self.logs_client = boto3.client('logs', region_name=region)
    
    def create_log_group(self, log_group_name, retention_days=14):
        """Create a log group with retention policy"""
        try:
            # Create log group
            self.logs_client.create_log_group(logGroupName=log_group_name)
            
            # Set retention policy
            self.logs_client.put_retention_policy(
                logGroupName=log_group_name,
                retentionInDays=retention_days
            )
            
            print(f"Created log group: {log_group_name}")
        except self.logs_client.exceptions.ResourceAlreadyExistsException:
            print(f"Log group already exists: {log_group_name}")
        except Exception as e:
            print(f"Error creating log group: {e}")
    
    def create_log_stream(self, log_group_name, log_stream_name):
        """Create a log stream"""
        try:
            self.logs_client.create_log_stream(
                logGroupName=log_group_name,
                logStreamName=log_stream_name
            )
            print(f"Created log stream: {log_stream_name}")
        except self.logs_client.exceptions.ResourceAlreadyExistsException:
            print(f"Log stream already exists: {log_stream_name}")
        except Exception as e:
            print(f"Error creating log stream: {e}")
    
    def put_log_events(self, log_group_name, log_stream_name, messages):
        """Send log events to CloudWatch"""
        try:
            # Get sequence token
            response = self.logs_client.describe_log_streams(
                logGroupName=log_group_name,
                logStreamNamePrefix=log_stream_name
            )
            
            sequence_token = None
            if response['logStreams']:
                sequence_token = response['logStreams'][0].get('uploadSequenceToken')
            
            # Prepare log events
            log_events = []
            for message in messages:
                log_events.append({
                    'timestamp': int(datetime.utcnow().timestamp() * 1000),
                    'message': message
                })
            
            # Sort by timestamp
            log_events.sort(key=lambda x: x['timestamp'])
            
            # Put log events
            put_args = {
                'logGroupName': log_group_name,
                'logStreamName': log_stream_name,
                'logEvents': log_events
            }
            
            if sequence_token:
                put_args['sequenceToken'] = sequence_token
            
            response = self.logs_client.put_log_events(**put_args)
            return response
            
        except Exception as e:
            print(f"Error putting log events: {e}")
            return None
    
    def query_logs(self, log_group_name, query_string, start_time, end_time):
        """Query logs using CloudWatch Logs Insights"""
        try:
            # Start query
            response = self.logs_client.start_query(
                logGroupName=log_group_name,
                startTime=int(start_time.timestamp()),
                endTime=int(end_time.timestamp()),
                queryString=query_string
            )
            
            query_id = response['queryId']
            
            # Wait for query to complete
            import time
            while True:
                result = self.logs_client.get_query_results(queryId=query_id)
                if result['status'] == 'Complete':
                    return result['results']
                elif result['status'] == 'Failed':
                    print(f"Query failed: {result}")
                    return None
                time.sleep(1)
                
        except Exception as e:
            print(f"Error querying logs: {e}")
            return None

# Usage
logs = CloudWatchLogs()

# Create log group and stream
logs.create_log_group('/aws/device/fire-tv', retention_days=30)
logs.create_log_stream('/aws/device/fire-tv', 'device-001')

# Send log events
messages = [
    "Device started successfully",
    "Network connection established",
    "Application launched: Netflix"
]
logs.put_log_events('/aws/device/fire-tv', 'device-001', messages)
```

### CloudWatch Logs Insights Queries
```sql
-- Find errors in the last hour
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

-- Count errors by hour
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() by bin(5m)

-- Find slow requests
fields @timestamp, @requestId, @duration
| filter @duration > 1000
| sort @duration desc

-- Parse JSON logs
fields @timestamp, @message
| filter @type = "REQUEST"
| parse @message "RequestId: * Duration: * ms"
| stats avg(Duration) by bin(5m)

-- Find unique IP addresses
fields @timestamp, @message
| parse @message /(?<ip>\d+\.\d+\.\d+\.\d+)/
| stats count() by ip
| sort count desc

-- Device-specific queries
fields @timestamp, device_id, cpu_usage, memory_usage
| filter device_id = "fire-tv-001"
| filter cpu_usage > 80
| sort @timestamp desc

-- Application performance
fields @timestamp, app_name, response_time
| filter app_name = "Netflix"
| stats avg(response_time), max(response_time), min(response_time) by bin(1h)
```

### Log Subscription Filters
```python
def create_subscription_filter():
    """Create subscription filter to send logs to Lambda"""
    logs_client = boto3.client('logs')
    
    logs_client.put_subscription_filter(
        logGroupName='/aws/device/fire-tv',
        filterName='ErrorFilter',
        filterPattern='ERROR',
        destinationArn='arn:aws:lambda:us-east-1:123456789012:function:ProcessLogs'
    )

def create_kinesis_subscription():
    """Create subscription filter to send logs to Kinesis"""
    logs_client = boto3.client('logs')
    
    logs_client.put_subscription_filter(
        logGroupName='/aws/device/fire-tv',
        filterName='AllLogsToKinesis',
        filterPattern='',  # All logs
        destinationArn='arn:aws:kinesis:us-east-1:123456789012:stream/log-stream',
        roleArn='arn:aws:iam::123456789012:role/CloudWatchLogsRole'
    )
```

---

## Alarms and Notifications

### Creating Alarms
```python
def create_cpu_alarm():
    """Create CPU utilization alarm"""
    cloudwatch = boto3.client('cloudwatch')
    
    cloudwatch.put_metric_alarm(
        AlarmName='HighCPUUtilization',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=2,
        MetricName='CPUUtilization',
        Namespace='AWS/EC2',
        Period=300,
        Statistic='Average',
        Threshold=80.0,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:cpu-alerts'
        ],
        AlarmDescription='Alarm when server CPU exceeds 80%',
        Dimensions=[
            {
                'Name': 'InstanceId',
                'Value': 'i-1234567890abcdef0'
            }
        ],
        Unit='Percent'
    )

def create_composite_alarm():
    """Create composite alarm based on multiple conditions"""
    cloudwatch = boto3.client('cloudwatch')
    
    cloudwatch.put_composite_alarm(
        AlarmName='SystemHealthAlarm',
        AlarmRule=(
            "ALARM('HighCPUUtilization') OR "
            "ALARM('HighMemoryUtilization') OR "
            "ALARM('DiskSpaceLow')"
        ),
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:system-alerts'
        ],
        AlarmDescription='Composite alarm for overall system health'
    )

def create_log_metric_alarm():
    """Create alarm based on log metrics"""
    logs_client = boto3.client('logs')
    cloudwatch = boto3.client('cloudwatch')
    
    # First create metric filter
    logs_client.put_metric_filter(
        logGroupName='/aws/device/fire-tv',
        filterName='ErrorCount',
        filterPattern='ERROR',
        metricTransformations=[
            {
                'metricName': 'ErrorCount',
                'metricNamespace': 'Device/Logs',
                'metricValue': '1',
                'defaultValue': 0
            }
        ]
    )
    
    # Then create alarm on the metric
    cloudwatch.put_metric_alarm(
        AlarmName='HighErrorRate',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=1,
        MetricName='ErrorCount',
        Namespace='Device/Logs',
        Period=300,
        Statistic='Sum',
        Threshold=5.0,
        ActionsEnabled=True,
        AlarmActions=[
            'arn:aws:sns:us-east-1:123456789012:error-alerts'
        ],
        AlarmDescription='Alarm when error count exceeds 5 in 5 minutes'
    )
```

### SNS Integration
```python
def setup_sns_notifications():
    """Setup SNS topic and subscriptions for alerts"""
    sns = boto3.client('sns')
    
    # Create SNS topic
    response = sns.create_topic(Name='device-alerts')
    topic_arn = response['TopicArn']
    
    # Subscribe email
    sns.subscribe(
        TopicArn=topic_arn,
        Protocol='email',
        Endpoint='support@company.com'
    )
    
    # Subscribe SMS
    sns.subscribe(
        TopicArn=topic_arn,
        Protocol='sms',
        Endpoint='+1234567890'
    )
    
    # Subscribe Lambda function
    sns.subscribe(
        TopicArn=topic_arn,
        Protocol='lambda',
        Endpoint='arn:aws:lambda:us-east-1:123456789012:function:ProcessAlert'
    )
    
    return topic_arn

def create_custom_notification_handler():
    """Lambda function to handle custom notifications"""
    import json
    
    def lambda_handler(event, context):
        # Parse SNS message
        for record in event['Records']:
            sns_message = json.loads(record['Sns']['Message'])
            
            alarm_name = sns_message['AlarmName']
            new_state = sns_message['NewStateValue']
            reason = sns_message['NewStateReason']
            
            # Custom notification logic
            if new_state == 'ALARM':
                send_slack_notification(alarm_name, reason)
                create_incident_ticket(alarm_name, reason)
            elif new_state == 'OK':
                close_incident_ticket(alarm_name)
        
        return {'statusCode': 200}
    
    def send_slack_notification(alarm_name, reason):
        # Implement Slack webhook
        pass
    
    def create_incident_ticket(alarm_name, reason):
        # Implement ticket creation
        pass
    
    def close_incident_ticket(alarm_name):
        # Implement ticket closure
        pass
```

---

## Dashboards

### Creating Dashboards
```python
def create_device_dashboard():
    """Create CloudWatch dashboard for device monitoring"""
    cloudwatch = boto3.client('cloudwatch')
    
    dashboard_body = {
        "widgets": [
            {
                "type": "metric",
                "x": 0,
                "y": 0,
                "width": 12,
                "height": 6,
                "properties": {
                    "metrics": [
                        ["AWS/EC2", "CPUUtilization", "InstanceId", "i-1234567890abcdef0"],
                        [".", "NetworkIn", ".", "."],
                        [".", "NetworkOut", ".", "."]
                    ],
                    "period": 300,
                    "stat": "Average",
                    "region": "us-east-1",
                    "title": "EC2 Instance Metrics"
                }
            },
            {
                "type": "log",
                "x": 0,
                "y": 6,
                "width": 24,
                "height": 6,
                "properties": {
                    "query": "SOURCE '/aws/device/fire-tv'\n| fields @timestamp, @message\n| filter @message like /ERROR/\n| sort @timestamp desc\n| limit 100",
                    "region": "us-east-1",
                    "title": "Recent Errors"
                }
            },
            {
                "type": "number",
                "x": 12,
                "y": 0,
                "width": 6,
                "height": 3,
                "properties": {
                    "metrics": [
                        ["Amazon/Devices", "ActiveDevices"]
                    ],
                    "period": 300,
                    "stat": "Average",
                    "region": "us-east-1",
                    "title": "Active Devices"
                }
            }
        ]
    }
    
    cloudwatch.put_dashboard(
        DashboardName='DeviceMonitoring',
        DashboardBody=json.dumps(dashboard_body)
    )

def create_application_dashboard():
    """Create dashboard for application performance"""
    dashboard_body = {
        "widgets": [
            {
                "type": "metric",
                "x": 0,
                "y": 0,
                "width": 12,
                "height": 6,
                "properties": {
                    "metrics": [
                        ["Amazon/Devices", "ResponseTime", "ApplicationName", "Netflix"],
                        [".", ".", ".", "Prime Video"],
                        [".", ".", ".", "Spotify"]
                    ],
                    "period": 300,
                    "stat": "Average",
                    "region": "us-east-1",
                    "title": "Application Response Times",
                    "yAxis": {
                        "left": {
                            "min": 0,
                            "max": 1000
                        }
                    }
                }
            },
            {
                "type": "metric",
                "x": 12,
                "y": 0,
                "width": 12,
                "height": 6,
                "properties": {
                    "metrics": [
                        ["Amazon/Devices", "ErrorCount", "ApplicationName", "Netflix"],
                        [".", ".", ".", "Prime Video"],
                        [".", ".", ".", "Spotify"]
                    ],
                    "period": 300,
                    "stat": "Sum",
                    "region": "us-east-1",
                    "title": "Application Error Counts"
                }
            }
        ]
    }
    
    cloudwatch.put_dashboard(
        DashboardName='ApplicationPerformance',
        DashboardBody=json.dumps(dashboard_body)
    )
```

---

## CloudWatch Agent

### Installation and Configuration
```bash
# Install CloudWatch Agent on EC2 (Amazon Linux 2)
sudo yum update -y
sudo yum install -y amazon-cloudwatch-agent

# Install on Ubuntu
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb
```

### Agent Configuration
```json
{
    "agent": {
        "metrics_collection_interval": 60,
        "run_as_user": "cwagent"
    },
    "metrics": {
        "namespace": "CWAgent",
        "metrics_collected": {
            "cpu": {
                "measurement": [
                    "cpu_usage_idle",
                    "cpu_usage_iowait",
                    "cpu_usage_user",
                    "cpu_usage_system"
                ],
                "metrics_collection_interval": 60,
                "totalcpu": false
            },
            "disk": {
                "measurement": [
                    "used_percent"
                ],
                "metrics_collection_interval": 60,
                "resources": [
                    "*"
                ]
            },
            "diskio": {
                "measurement": [
                    "io_time",
                    "read_bytes",
                    "write_bytes",
                    "reads",
                    "writes"
                ],
                "metrics_collection_interval": 60,
                "resources": [
                    "*"
                ]
            },
            "mem": {
                "measurement": [
                    "mem_used_percent"
                ],
                "metrics_collection_interval": 60
            },
            "netstat": {
                "measurement": [
                    "tcp_established",
                    "tcp_time_wait"
                ],
                "metrics_collection_interval": 60
            },
            "swap": {
                "measurement": [
                    "swap_used_percent"
                ],
                "metrics_collection_interval": 60
            }
        }
    },
    "logs": {
        "logs_collected": {
            "files": {
                "collect_list": [
                    {
                        "file_path": "/var/log/messages",
                        "log_group_name": "/aws/ec2/var/log/messages",
                        "log_stream_name": "{instance_id}",
                        "timezone": "UTC"
                    },
                    {
                        "file_path": "/var/log/secure",
                        "log_group_name": "/aws/ec2/var/log/secure",
                        "log_stream_name": "{instance_id}",
                        "timezone": "UTC"
                    },
                    {
                        "file_path": "/var/log/application/*.log",
                        "log_group_name": "/aws/application/logs",
                        "log_stream_name": "{instance_id}/{hostname}",
                        "timezone": "UTC",
                        "multi_line_start_pattern": "^\\d{4}-\\d{2}-\\d{2}"
                    }
                ]
            }
        }
    }
}
```

### Managing CloudWatch Agent
```bash
# Start the agent with configuration
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
    -s

# Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -m ec2 \
    -a query

# Stop the agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -m ec2 \
    -a stop

# View agent logs
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

---

## Device Monitoring

### Device-Specific Metrics
```python
class DeviceMonitoring:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.namespace = 'Amazon/Devices'
    
    def send_device_metrics(self, device_id, device_type, metrics_data):
        """Send comprehensive device metrics"""
        
        base_dimensions = [
            {'Name': 'DeviceId', 'Value': device_id},
            {'Name': 'DeviceType', 'Value': device_type}
        ]
        
        metric_data = []
        
        # System metrics
        if 'system' in metrics_data:
            system = metrics_data['system']
            system_metrics = [
                ('CPUUtilization', system.get('cpu_percent', 0), 'Percent'),
                ('MemoryUtilization', system.get('memory_percent', 0), 'Percent'),
                ('DiskUtilization', system.get('disk_percent', 0), 'Percent'),
                ('Temperature', system.get('temperature', 0), 'None'),
                ('BatteryLevel', system.get('battery_level', 0), 'Percent')
            ]
            
            for name, value, unit in system_metrics:
                metric_data.append({
                    'MetricName': name,
                    'Value': value,
                    'Unit': unit,
                    'Dimensions': base_dimensions
                })
        
        # Network metrics
        if 'network' in metrics_data:
            network = metrics_data['network']
            network_metrics = [
                ('NetworkLatency', network.get('latency', 0), 'Milliseconds'),
                ('NetworkThroughput', network.get('throughput', 0), 'Bytes/Second'),
                ('PacketLoss', network.get('packet_loss', 0), 'Percent'),
                ('SignalStrength', network.get('signal_strength', 0), 'None')
            ]
            
            for name, value, unit in network_metrics:
                metric_data.append({
                    'MetricName': name,
                    'Value': value,
                    'Unit': unit,
                    'Dimensions': base_dimensions + [
                        {'Name': 'NetworkType', 'Value': network.get('type', 'WiFi')}
                    ]
                })
        
        # Application metrics
        if 'applications' in metrics_data:
            for app_name, app_data in metrics_data['applications'].items():
                app_dimensions = base_dimensions + [
                    {'Name': 'ApplicationName', 'Value': app_name}
                ]
                
                app_metrics = [
                    ('ResponseTime', app_data.get('response_time', 0), 'Milliseconds'),
                    ('ErrorCount', app_data.get('error_count', 0), 'Count'),
                    ('RequestCount', app_data.get('request_count', 0), 'Count'),
                    ('MemoryUsage', app_data.get('memory_usage', 0), 'Bytes')
                ]
                
                for name, value, unit in app_metrics:
                    metric_data.append({
                        'MetricName': name,
                        'Value': value,
                        'Unit': unit,
                        'Dimensions': app_dimensions
                    })
        
        # Send metrics in batches (CloudWatch limit is 20 metrics per call)
        batch_size = 20
        for i in range(0, len(metric_data), batch_size):
            batch = metric_data[i:i + batch_size]
            try:
                self.cloudwatch.put_metric_data(
                    Namespace=self.namespace,
                    MetricData=batch
                )
            except Exception as e:
                print(f"Error sending metric batch: {e}")

# Usage example
device_monitor = DeviceMonitoring()
metrics_data = {
    'system': {
        'cpu_percent': 45.2,
        'memory_percent': 67.8,
        'disk_percent': 23.1,
        'temperature': 42.5,
        'battery_level': 85
    },
    'network': {
        'latency': 15.3,
        'throughput': 1024000,
        'packet_loss': 0.1,
        'signal_strength': -45,
        'type': 'WiFi'
    },
    'applications': {
        'Netflix': {
            'response_time': 250,
            'error_count': 0,
            'request_count': 150,
            'memory_usage': 512000000
        },
        'Prime Video': {
            'response_time': 180,
            'error_count': 2,
            'request_count': 89,
            'memory_usage': 384000000
        }
    }
}

device_monitor.send_device_metrics('fire-tv-001', 'Fire TV', metrics_data)
```

---

## Cost Optimization

### Understanding CloudWatch Costs
```bash
# Cost Components
1. Metrics: $0.30 per metric per month (first 10,000 metrics free)
2. API Requests: $0.01 per 1,000 requests
3. Logs Ingestion: $0.50 per GB ingested
4. Logs Storage: $0.03 per GB per month
5. Logs Insights Queries: $0.005 per GB scanned
6. Dashboards: $3.00 per dashboard per month
7. Alarms: $0.10 per alarm per month (first 10 alarms free)
```

### Cost Optimization Strategies
```python
class CostOptimizer:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.logs_client = boto3.client('logs')
    
    def optimize_log_retention(self, log_groups_config):
        """Set appropriate retention periods for log groups"""
        for log_group, retention_days in log_groups_config.items():
            try:
                self.logs_client.put_retention_policy(
                    logGroupName=log_group,
                    retentionInDays=retention_days
                )
                print(f"Set retention for {log_group}: {retention_days} days")
            except Exception as e:
                print(f"Error setting retention for {log_group}: {e}")
    
    def cleanup_unused_log_groups(self, days_inactive=30):
        """Delete log groups with no recent activity"""
        cutoff_time = datetime.now() - timedelta(days=days_inactive)
        
        paginator = self.logs_client.get_paginator('describe_log_groups')
        
        for page in paginator.paginate():
            for log_group in page['logGroups']:
                last_event_time = log_group.get('lastEventTime', 0)
                if last_event_time < cutoff_time.timestamp() * 1000:
                    print(f"Inactive log group: {log_group['logGroupName']}")
                    # Uncomment to actually delete
                    # self.logs_client.delete_log_group(
                    #     logGroupName=log_group['logGroupName']
                    # )
    
    def optimize_metric_frequency(self, metric_configs):
        """Reduce metric collection frequency for non-critical metrics"""
        for config in metric_configs:
            namespace = config['namespace']
            metric_name = config['metric_name']
            new_period = config['period']  # Increase period to reduce costs
            
            print(f"Consider increasing period for {namespace}/{metric_name} to {new_period}s")
    
    def estimate_costs(self, metrics_count, log_gb_per_month, alarms_count, dashboards_count):
        """Estimate monthly CloudWatch costs"""
        # Metrics cost (first 10,000 free)
        billable_metrics = max(0, metrics_count - 10000)
        metrics_cost = billable_metrics * 0.30
        
        # Logs cost
        logs_ingestion_cost = log_gb_per_month * 0.50
        logs_storage_cost = log_gb_per_month * 0.03
        
        # Alarms cost (first 10 free)
        billable_alarms = max(0, alarms_count - 10)
        alarms_cost = billable_alarms * 0.10
        
        # Dashboards cost
        dashboards_cost = dashboards_count * 3.00
        
        total_cost = (metrics_cost + logs_ingestion_cost + 
                     logs_storage_cost + alarms_cost + dashboards_cost)
        
        return {
            'metrics_cost': metrics_cost,
            'logs_ingestion_cost': logs_ingestion_cost,
            'logs_storage_cost': logs_storage_cost,
            'alarms_cost': alarms_cost,
            'dashboards_cost': dashboards_cost,
            'total_monthly_cost': total_cost
        }

# Usage
optimizer = CostOptimizer()

# Set retention policies
log_retention_config = {
    '/aws/device/fire-tv': 30,      # 30 days for device logs
    '/aws/application/debug': 7,     # 7 days for debug logs
    '/aws/application/error': 90,    # 90 days for error logs
    '/aws/lambda/function': 14       # 14 days for Lambda logs
}
optimizer.optimize_log_retention(log_retention_config)

# Estimate costs
cost_estimate = optimizer.estimate_costs(
    metrics_count=5000,
    log_gb_per_month=100,
    alarms_count=25,
    dashboards_count=5
)
print(f"Estimated monthly cost: ${cost_estimate['total_monthly_cost']:.2f}")
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Metrics Not Appearing
```python
def troubleshoot_missing_metrics():
    """Troubleshoot missing metrics issues"""
    
    # Check IAM permissions
    required_permissions = [
        'cloudwatch:PutMetricData',
        'cloudwatch:GetMetricStatistics',
        'cloudwatch:ListMetrics'
    ]
    
    # Check metric namespace and dimensions
    def validate_metric_data(metric_data):
        for metric in metric_data:
            # Validate metric name
            if not metric.get('MetricName'):
                print("Error: MetricName is required")
            
            # Validate dimensions
            dimensions = metric.get('Dimensions', [])
            if len(dimensions) > 10:
                print("Error: Maximum 10 dimensions allowed")
            
            # Validate timestamp
            timestamp = metric.get('Timestamp')
            if timestamp:
                now = datetime.utcnow()
                if abs((now - timestamp).total_seconds()) > 14 * 24 * 3600:
                    print("Error: Timestamp must be within 14 days")
    
    # Check for throttling
    def check_api_throttling():
        try:
            cloudwatch = boto3.client('cloudwatch')
            response = cloudwatch.list_metrics(MaxRecords=1)
            print("API call successful - no throttling detected")
        except ClientError as e:
            if e.response['Error']['Code'] == 'Throttling':
                print("API throttling detected - reduce request rate")
            else:
                print(f"API error: {e}")

#### 2. Log Events Not Appearing
```python
def troubleshoot_log_issues():
    """Troubleshoot CloudWatch Logs issues"""
    
    # Check log group and stream existence
    def verify_log_resources(log_group_name, log_stream_name):
        logs_client = boto3.client('logs')
        
        try:
            # Check log group
            logs_client.describe_log_groups(
                logGroupNamePrefix=log_group_name
            )
            print(f"Log group exists: {log_group_name}")
            
            # Check log stream
            response = logs_client.describe_log_streams(
                logGroupName=log_group_name,
                logStreamNamePrefix=log_stream_name
            )
            
            if response['logStreams']:
                print(f"Log stream exists: {log_stream_name}")
            else:
                print(f"Log stream not found: {log_stream_name}")
                
        except ClientError as e:
            print(f"Error checking log resources: {e}")
    
    # Check sequence token
    def get_sequence_token(log_group_name, log_stream_name):
        logs_client = boto3.client('logs')
        
        try:
            response = logs_client.describe_log_streams(
                logGroupName=log_group_name,
                logStreamNamePrefix=log_stream_name
            )
            
            if response['logStreams']:
                stream = response['logStreams'][0]
                return stream.get('uploadSequenceToken')
            
        except ClientError as e:
            print(f"Error getting sequence token: {e}")
        
        return None

#### 3. High CloudWatch Costs
```python
def analyze_cloudwatch_costs():
    """Analyze and optimize CloudWatch costs"""
    
    # Find expensive log groups
    def find_expensive_log_groups():
        logs_client = boto3.client('logs')
        
        paginator = logs_client.get_paginator('describe_log_groups')
        expensive_groups = []
        
        for page in paginator.paginate():
            for log_group in page['logGroups']:
                stored_bytes = log_group.get('storedBytes', 0)
                if stored_bytes > 1024 * 1024 * 1024:  # > 1GB
                    expensive_groups.append({
                        'name': log_group['logGroupName'],
                        'size_gb': stored_bytes / (1024 * 1024 * 1024),
                        'retention': log_group.get('retentionInDays', 'Never expire')
                    })
        
        return sorted(expensive_groups, key=lambda x: x['size_gb'], reverse=True)
    
    # Find unused metrics
    def find_unused_metrics():
        cloudwatch = boto3.client('cloudwatch')
        
        # Get all custom metrics
        paginator = cloudwatch.get_paginator('list_metrics')
        custom_metrics = []
        
        for page in paginator.paginate():
            for metric in page['Metrics']:
                if not metric['Namespace'].startswith('AWS/'):
                    custom_metrics.append(metric)
        
        # Check which metrics have recent data
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(days=7)
        
        unused_metrics = []
        for metric in custom_metrics:
            try:
                response = cloudwatch.get_metric_statistics(
                    Namespace=metric['Namespace'],
                    MetricName=metric['MetricName'],
                    Dimensions=metric['Dimensions'],
                    StartTime=start_time,
                    EndTime=end_time,
                    Period=3600,
                    Statistics=['Average']
                )
                
                if not response['Datapoints']:
                    unused_metrics.append(metric)
                    
            except Exception as e:
                print(f"Error checking metric {metric['MetricName']}: {e}")
        
        return unused_metrics
```

---

## CLI and API Usage

### AWS CLI Commands
```bash
# Metrics
aws cloudwatch put-metric-data \
    --namespace "Amazon/Devices" \
    --metric-data MetricName=CPUUtilization,Value=75.5,Unit=Percent,Dimensions=[{Name=DeviceId,Value=fire-tv-001}]

aws cloudwatch get-metric-statistics \
    --namespace "AWS/EC2" \
    --metric-name CPUUtilization \
    --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
    --start-time 2023-10-11T00:00:00Z \
    --end-time 2023-10-11T23:59:59Z \
    --period 3600 \
    --statistics Average,Maximum

# Logs
aws logs create-log-group --log-group-name "/aws/device/fire-tv"

aws logs put-log-events \
    --log-group-name "/aws/device/fire-tv" \
    --log-stream-name "device-001" \
    --log-events timestamp=1697000000000,message="Device started"

aws logs start-query \
    --log-group-name "/aws/device/fire-tv" \
    --start-time 1697000000 \
    --end-time 1697086400 \
    --query-string 'fields @timestamp, @message | filter @message like /ERROR/'

# Alarms
aws cloudwatch put-metric-alarm \
    --alarm-name "HighCPU" \
    --alarm-description "High CPU utilization" \
    --metric-name CPUUtilization \
    --namespace AWS/EC2 \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2

# Dashboards
aws cloudwatch put-dashboard \
    --dashboard-name "DeviceMonitoring" \
    --dashboard-body file://dashboard.json
```

### Python SDK Examples
```python
import boto3
from datetime import datetime, timedelta

# Initialize clients
cloudwatch = boto3.client('cloudwatch')
logs_client = boto3.client('logs')

# Send custom metrics
def send_batch_metrics(metrics_data):
    """Send multiple metrics efficiently"""
    
    metric_data = []
    for metric in metrics_data:
        metric_data.append({
            'MetricName': metric['name'],
            'Value': metric['value'],
            'Unit': metric.get('unit', 'Count'),
            'Timestamp': datetime.utcnow(),
            'Dimensions': metric.get('dimensions', [])
        })
    
    # Send in batches of 20 (CloudWatch limit)
    batch_size = 20
    for i in range(0, len(metric_data), batch_size):
        batch = metric_data[i:i + batch_size]
        
        try:
            cloudwatch.put_metric_data(
                Namespace='Amazon/Devices',
                MetricData=batch
            )
            print(f"Sent batch of {len(batch)} metrics")
        except Exception as e:
            print(f"Error sending metrics: {e}")

# Query logs programmatically
def query_device_errors(device_id, hours_back=24):
    """Query device errors from logs"""
    
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=hours_back)
    
    query_string = f"""
    fields @timestamp, @message
    | filter device_id = "{device_id}"
    | filter @message like /ERROR/
    | sort @timestamp desc
    | limit 100
    """
    
    try:
        # Start query
        response = logs_client.start_query(
            logGroupName='/aws/device/fire-tv',
            startTime=int(start_time.timestamp()),
            endTime=int(end_time.timestamp()),
            queryString=query_string
        )
        
        query_id = response['queryId']
        
        # Wait for completion
        import time
        while True:
            result = logs_client.get_query_results(queryId=query_id)
            
            if result['status'] == 'Complete':
                return result['results']
            elif result['status'] == 'Failed':
                print(f"Query failed: {result}")
                return None
            
            time.sleep(1)
            
    except Exception as e:
        print(f"Error querying logs: {e}")
        return None

# Create comprehensive monitoring setup
def setup_device_monitoring(device_id, device_type):
    """Set up complete monitoring for a device"""
    
    # Create log group
    log_group_name = f'/aws/device/{device_type.lower()}'
    try:
        logs_client.create_log_group(logGroupName=log_group_name)
        logs_client.put_retention_policy(
            logGroupName=log_group_name,
            retentionInDays=30
        )
    except logs_client.exceptions.ResourceAlreadyExistsException:
        pass
    
    # Create log stream
    log_stream_name = device_id
    try:
        logs_client.create_log_stream(
            logGroupName=log_group_name,
            logStreamName=log_stream_name
        )
    except logs_client.exceptions.ResourceAlreadyExistsException:
        pass
    
    # Create alarms
    alarms = [
        {
            'name': f'{device_id}-HighCPU',
            'metric': 'CPUUtilization',
            'threshold': 80,
            'comparison': 'GreaterThanThreshold'
        },
        {
            'name': f'{device_id}-HighMemory',
            'metric': 'MemoryUtilization',
            'threshold': 85,
            'comparison': 'GreaterThanThreshold'
        },
        {
            'name': f'{device_id}-HighLatency',
            'metric': 'NetworkLatency',
            'threshold': 100,
            'comparison': 'GreaterThanThreshold'
        }
    ]
    
    for alarm in alarms:
        cloudwatch.put_metric_alarm(
            AlarmName=alarm['name'],
            ComparisonOperator=alarm['comparison'],
            EvaluationPeriods=2,
            MetricName=alarm['metric'],
            Namespace='Amazon/Devices',
            Period=300,
            Statistic='Average',
            Threshold=alarm['threshold'],
            ActionsEnabled=True,
            AlarmDescription=f'Alarm for {device_id} {alarm["metric"]}',
            Dimensions=[
                {
                    'Name': 'DeviceId',
                    'Value': device_id
                },
                {
                    'Name': 'DeviceType',
                    'Value': device_type
                }
            ]
        )
    
    print(f"Monitoring setup complete for {device_id}")
```

---

## Interview Scenarios

### Scenario 1: Device Performance Monitoring
**Question**: "How would you set up monitoring for Amazon Fire TV devices to track performance and detect issues proactively?"

**Answer Approach**:
```python
# 1. Define key metrics to monitor
key_metrics = [
    'CPUUtilization',      # System performance
    'MemoryUtilization',   # Memory usage
    'NetworkLatency',      # Connectivity
    'ApplicationResponseTime',  # App performance
    'ErrorCount',          # Application errors
    'Temperature',         # Hardware health
    'BufferingEvents'      # Streaming quality
]

# 2. Set up custom metrics collection
def setup_fire_tv_monitoring():
    # Create namespace for Fire TV metrics
    namespace = 'Amazon/FireTV'
    
    # Define dimensions for filtering
    dimensions = [
        'DeviceId',      # Unique device identifier
        'DeviceModel',   # Fire TV model
        'SoftwareVersion', # OS version
        'Location',      # Geographic region
        'NetworkType'    # WiFi/Ethernet
    ]
    
    # Set up alarms with appropriate thresholds
    alarms = {
        'HighCPU': {'threshold': 80, 'period': 300},
        'HighMemory': {'threshold': 85, 'period': 300},
        'HighLatency': {'threshold': 100, 'period': 300},
        'HighErrorRate': {'threshold': 5, 'period': 300}
    }

# 3. Create dashboards for visualization
def create_fire_tv_dashboard():
    # System health overview
    # Application performance metrics
    # Network connectivity status
    # Error rate trends
    # Geographic distribution of issues
```

### Scenario 2: Log Analysis for Troubleshooting
**Question**: "A customer reports that their Fire TV is freezing during Netflix playback. How would you use CloudWatch Logs to investigate?"

**Answer Approach**:
```sql
-- 1. Find device-specific logs
fields @timestamp, @message, device_id, app_name
| filter device_id = "customer-device-id"
| filter app_name = "Netflix"
| sort @timestamp desc
| limit 100

-- 2. Look for error patterns around the time of issue
fields @timestamp, @message
| filter device_id = "customer-device-id"
| filter @timestamp >= "2023-10-11T14:00:00"
| filter @timestamp <= "2023-10-11T15:00:00"
| filter @message like /ERROR|FREEZE|CRASH|EXCEPTION/
| sort @timestamp asc

-- 3. Check system resource usage
fields @timestamp, cpu_usage, memory_usage, temperature
| filter device_id = "customer-device-id"
| filter @timestamp >= "2023-10-11T14:00:00"
| filter cpu_usage > 90 or memory_usage > 90
| sort @timestamp asc

-- 4. Analyze network connectivity
fields @timestamp, network_latency, packet_loss, bandwidth
| filter device_id = "customer-device-id"
| filter @timestamp >= "2023-10-11T14:00:00"
| stats avg(network_latency), max(packet_loss) by bin(5m)
```

### Scenario 3: Cost Optimization
**Question**: "Your CloudWatch costs have increased significantly. How would you identify and reduce the costs while maintaining monitoring effectiveness?"

**Answer Approach**:
```python
# 1. Analyze current usage and costs
def analyze_cloudwatch_costs():
    # Identify high-volume log groups
    # Find unused or low-value metrics
    # Check alarm utilization
    # Review dashboard usage
    
    cost_analysis = {
        'logs_ingestion': 'Check log volume by group',
        'logs_storage': 'Review retention policies',
        'metrics': 'Identify unused custom metrics',
        'alarms': 'Remove redundant alarms',
        'dashboards': 'Consolidate or remove unused dashboards'
    }

# 2. Optimization strategies
optimization_plan = {
    'immediate': [
        'Reduce log retention for debug logs (7 days)',
        'Delete unused log groups',
        'Remove redundant alarms',
        'Consolidate similar dashboards'
    ],
    'medium_term': [
        'Implement log sampling for high-volume apps',
        'Use metric filters instead of custom metrics where possible',
        'Optimize metric collection frequency',
        'Implement log aggregation before sending to CloudWatch'
    ],
    'long_term': [
        'Consider alternative storage for long-term logs',
        'Implement intelligent alerting to reduce noise',
        'Use composite alarms to reduce alarm count',
        'Implement cost monitoring and budgets'
    ]
}

# 3. Implementation priorities
def prioritize_optimizations():
    # High impact, low effort: Retention policies, unused resources
    # Medium impact, medium effort: Metric optimization, log sampling
    # High impact, high effort: Architecture changes, alternative solutions
```

### Key Interview Points to Remember:
1. **Systematic Approach**: Always explain your methodology
2. **Cost Awareness**: Understand CloudWatch pricing model
3. **Scalability**: Design for growth and multiple devices
4. **Automation**: Emphasize automated monitoring and alerting
5. **Integration**: Show how CloudWatch fits into broader AWS ecosystem
6. **Troubleshooting**: Demonstrate log analysis and correlation skills
7. **Best Practices**: Follow AWS recommended practices for monitoring
