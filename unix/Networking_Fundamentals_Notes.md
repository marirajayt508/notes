# Networking Fundamentals Notes - Amazon Lab126 Support Engineer

## Table of Contents
1. [Network Basics](#network-basics)
2. [TCP/IP Protocol Suite](#tcpip-protocol-suite)
3. [Network Troubleshooting](#network-troubleshooting)
4. [DNS and DHCP](#dns-and-dhcp)
5. [Wireless Networking](#wireless-networking)
6. [Network Security](#network-security)
7. [Network Monitoring](#network-monitoring)
8. [Device Connectivity](#device-connectivity)
9. [Performance Analysis](#performance-analysis)
10. [Common Issues and Solutions](#common-issues-and-solutions)
11. [Network Tools and Commands](#network-tools-and-commands)
12. [Interview Scenarios](#interview-scenarios)

---

## Network Basics

### OSI Model
```
Layer 7: Application    (HTTP, HTTPS, FTP, SMTP, DNS)
Layer 6: Presentation   (SSL/TLS, encryption, compression)
Layer 5: Session        (NetBIOS, RPC, SQL sessions)
Layer 4: Transport      (TCP, UDP)
Layer 3: Network        (IP, ICMP, routing)
Layer 2: Data Link      (Ethernet, WiFi, switches)
Layer 1: Physical       (Cables, hubs, repeaters)
```

### TCP/IP Model
```
Application Layer    (HTTP, HTTPS, FTP, SMTP, DNS, DHCP)
Transport Layer      (TCP, UDP)
Internet Layer       (IP, ICMP, ARP)
Network Access       (Ethernet, WiFi)
```

### IP Addressing
```bash
# IPv4 Address Classes
Class A: 1.0.0.0    - 126.255.255.255  (/8)  - 16,777,214 hosts
Class B: 128.0.0.0  - 191.255.255.255  (/16) - 65,534 hosts
Class C: 192.0.0.0  - 223.255.255.255  (/24) - 254 hosts

# Private IP Ranges (RFC 1918)
Class A: 10.0.0.0/8         (10.0.0.0 - 10.255.255.255)
Class B: 172.16.0.0/12      (172.16.0.0 - 172.31.255.255)
Class C: 192.168.0.0/16     (192.168.0.0 - 192.168.255.255)

# Special Addresses
127.0.0.1           # Loopback
0.0.0.0             # Default route
255.255.255.255     # Broadcast
169.254.0.0/16      # Link-local (APIPA)
```

### Subnetting
```bash
# CIDR Notation Examples
192.168.1.0/24      # Subnet mask: 255.255.255.0 (254 hosts)
192.168.1.0/25      # Subnet mask: 255.255.255.128 (126 hosts)
192.168.1.0/26      # Subnet mask: 255.255.255.192 (62 hosts)
192.168.1.0/27      # Subnet mask: 255.255.255.224 (30 hosts)

# Subnet Calculation
Network: 192.168.1.0/26
Subnet Mask: 255.255.255.192
Network Address: 192.168.1.0
Broadcast Address: 192.168.1.63
Usable IPs: 192.168.1.1 - 192.168.1.62
```

---

## TCP/IP Protocol Suite

### Internet Protocol (IP)
```bash
# IPv4 Header Fields
Version (4 bits)        # IP version (4 or 6)
Header Length (4 bits)  # Header length in 32-bit words
Type of Service (8 bits) # QoS and priority
Total Length (16 bits)  # Total packet length
Identification (16 bits) # Fragment identification
Flags (3 bits)          # Fragmentation flags
Fragment Offset (13 bits) # Fragment position
TTL (8 bits)            # Time to Live
Protocol (8 bits)       # Next layer protocol (TCP=6, UDP=17, ICMP=1)
Header Checksum (16 bits) # Error detection
Source Address (32 bits)  # Source IP
Destination Address (32 bits) # Destination IP
```

### Transmission Control Protocol (TCP)
```bash
# TCP Header Fields
Source Port (16 bits)       # Source port number
Destination Port (16 bits)  # Destination port number
Sequence Number (32 bits)   # Data sequencing
Acknowledgment Number (32 bits) # Next expected sequence
Header Length (4 bits)      # TCP header length
Flags (9 bits)             # Control flags (SYN, ACK, FIN, RST, etc.)
Window Size (16 bits)       # Flow control
Checksum (16 bits)         # Error detection
Urgent Pointer (16 bits)   # Urgent data pointer

# TCP Connection States
LISTEN          # Waiting for connection request
SYN-SENT        # Connection request sent
SYN-RECEIVED    # Connection request received
ESTABLISHED     # Connection established
FIN-WAIT-1      # Connection termination initiated
FIN-WAIT-2      # Waiting for FIN from remote
CLOSE-WAIT      # Waiting for local application to close
CLOSING         # Both sides closing simultaneously
LAST-ACK        # Waiting for final ACK
TIME-WAIT       # Waiting for network to clear packets
CLOSED          # Connection closed
```

### User Datagram Protocol (UDP)
```bash
# UDP Header Fields
Source Port (16 bits)       # Source port number
Destination Port (16 bits)  # Destination port number
Length (16 bits)           # UDP header + data length
Checksum (16 bits)         # Error detection

# UDP Characteristics
- Connectionless protocol
- No reliability guarantees
- No flow control
- Lower overhead than TCP
- Used for: DNS, DHCP, streaming media, gaming
```

### Common Ports
```bash
# Well-Known Ports (0-1023)
20/21   FTP (Data/Control)
22      SSH
23      Telnet
25      SMTP
53      DNS
67/68   DHCP (Server/Client)
80      HTTP
110     POP3
143     IMAP
443     HTTPS
993     IMAPS
995     POP3S

# Registered Ports (1024-49151)
1433    Microsoft SQL Server
3306    MySQL
3389    RDP
5432    PostgreSQL
8080    HTTP Alternate

# Dynamic/Private Ports (49152-65535)
Used for client connections
```

---

## Network Troubleshooting

### Systematic Troubleshooting Approach
```bash
# Layer-by-Layer Troubleshooting
1. Physical Layer
   - Check cables and connections
   - Verify link lights on network interfaces
   - Test with different cables/ports

2. Data Link Layer
   - Check MAC address tables
   - Verify VLAN configuration
   - Check for duplex mismatches

3. Network Layer
   - Verify IP configuration
   - Check routing tables
   - Test with ping

4. Transport Layer
   - Check port connectivity
   - Verify firewall rules
   - Test with telnet/nc

5. Application Layer
   - Check application logs
   - Verify service status
   - Test application functionality
```

### Essential Network Commands
```bash
# Connectivity Testing
ping -c 4 8.8.8.8              # Test basic connectivity
ping6 -c 4 2001:4860:4860::8888 # IPv6 ping
traceroute google.com           # Trace route to destination
tracepath google.com            # Alternative traceroute
mtr google.com                  # Continuous traceroute

# Interface Configuration
ip addr show                    # Show IP addresses
ip link show                    # Show network interfaces
ifconfig                        # Legacy interface configuration
ip route show                   # Show routing table
route -n                        # Numeric routing table

# Network Statistics
netstat -tuln                   # Show listening ports
netstat -rn                     # Show routing table
ss -tuln                        # Modern netstat replacement
ss -tulpn                       # Include process names

# DNS Testing
nslookup google.com             # DNS lookup
dig google.com                  # Detailed DNS query
dig @8.8.8.8 google.com        # Query specific DNS server
host google.com                 # Simple DNS lookup

# Port Testing
telnet google.com 80            # Test TCP connection
nc -zv google.com 80            # Test port with netcat
nmap -p 80 google.com           # Port scan
```

### Network Troubleshooting Scripts
```python
#!/usr/bin/env python3
import subprocess
import socket
import time
from datetime import datetime

class NetworkTroubleshooter:
    def __init__(self):
        self.results = {}
    
    def test_connectivity(self, host, count=4):
        """Test basic connectivity with ping"""
        try:
            result = subprocess.run(
                ['ping', '-c', str(count), host],
                capture_output=True,
                text=True,
                timeout=30
            )
            
            success = result.returncode == 0
            self.results[f'ping_{host}'] = {
                'success': success,
                'output': result.stdout,
                'error': result.stderr
            }
            
            if success:
                # Parse ping statistics
                lines = result.stdout.split('\n')
                for line in lines:
                    if 'packet loss' in line:
                        loss = line.split(',')[2].strip().split()[0]
                        self.results[f'ping_{host}']['packet_loss'] = loss
                    elif 'min/avg/max' in line:
                        times = line.split('=')[1].strip().split('/')
                        self.results[f'ping_{host}']['latency'] = {
                            'min': float(times[0]),
                            'avg': float(times[1]),
                            'max': float(times[2])
                        }
            
            return success
        except Exception as e:
            self.results[f'ping_{host}'] = {
                'success': False,
                'error': str(e)
            }
            return False
    
    def test_port(self, host, port, timeout=5):
        """Test TCP port connectivity"""
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(timeout)
            result = sock.connect_ex((host, port))
            sock.close()
            
            success = result == 0
            self.results[f'port_{host}_{port}'] = {
                'success': success,
                'port': port,
                'host': host
            }
            
            return success
        except Exception as e:
            self.results[f'port_{host}_{port}'] = {
                'success': False,
                'error': str(e)
            }
            return False
    
    def test_dns(self, hostname):
        """Test DNS resolution"""
        try:
            ip = socket.gethostbyname(hostname)
            self.results[f'dns_{hostname}'] = {
                'success': True,
                'ip': ip,
                'hostname': hostname
            }
            return True, ip
        except socket.gaierror as e:
            self.results[f'dns_{hostname}'] = {
                'success': False,
                'error': str(e),
                'hostname': hostname
            }
            return False, None
    
    def traceroute(self, host):
        """Perform traceroute"""
        try:
            result = subprocess.run(
                ['traceroute', host],
                capture_output=True,
                text=True,
                timeout=60
            )
            
            self.results[f'traceroute_{host}'] = {
                'success': result.returncode == 0,
                'output': result.stdout,
                'error': result.stderr
            }
            
            return result.returncode == 0
        except Exception as e:
            self.results[f'traceroute_{host}'] = {
                'success': False,
                'error': str(e)
            }
            return False
    
    def check_local_network(self):
        """Check local network configuration"""
        try:
            # Get default gateway
            result = subprocess.run(
                ['ip', 'route', 'show', 'default'],
                capture_output=True,
                text=True
            )
            
            if result.returncode == 0:
                gateway_line = result.stdout.strip()
                if gateway_line:
                    gateway = gateway_line.split()[2]
                    self.results['default_gateway'] = gateway
                    
                    # Test gateway connectivity
                    self.test_connectivity(gateway, count=2)
            
            # Get local IP addresses
            result = subprocess.run(
                ['ip', 'addr', 'show'],
                capture_output=True,
                text=True
            )
            
            self.results['local_interfaces'] = result.stdout
            
        except Exception as e:
            self.results['local_network_error'] = str(e)
    
    def comprehensive_test(self, target_hosts=None):
        """Run comprehensive network tests"""
        if target_hosts is None:
            target_hosts = ['8.8.8.8', 'google.com', '1.1.1.1']
        
        print(f"Starting network diagnostics at {datetime.now()}")
        
        # Check local network
        print("Checking local network configuration...")
        self.check_local_network()
        
        # Test connectivity to various hosts
        for host in target_hosts:
            print(f"Testing connectivity to {host}...")
            self.test_connectivity(host)
            
            # Test DNS if it's a hostname
            if not host.replace('.', '').isdigit():
                print(f"Testing DNS resolution for {host}...")
                success, ip = self.test_dns(host)
                if success:
                    # Test connectivity to resolved IP
                    self.test_connectivity(ip, count=2)
        
        # Test common ports
        common_ports = [80, 443, 53, 22]
        for host in ['google.com', '8.8.8.8']:
            for port in common_ports:
                if host == '8.8.8.8' and port in [80, 443]:
                    continue  # Skip HTTP ports for DNS server
                print(f"Testing port {port} on {host}...")
                self.test_port(host, port)
        
        # Traceroute to external host
        print("Running traceroute...")
        self.traceroute('google.com')
        
        return self.results
    
    def generate_report(self):
        """Generate troubleshooting report"""
        report = []
        report.append("Network Troubleshooting Report")
        report.append("=" * 40)
        report.append(f"Generated: {datetime.now()}")
        report.append("")
        
        # Connectivity tests
        report.append("Connectivity Tests:")
        for key, result in self.results.items():
            if key.startswith('ping_'):
                host = key.replace('ping_', '')
                if result['success']:
                    latency = result.get('latency', {})
                    avg_latency = latency.get('avg', 'N/A')
                    packet_loss = result.get('packet_loss', 'N/A')
                    report.append(f"  ✓ {host}: Avg latency {avg_latency}ms, Loss {packet_loss}")
                else:
                    report.append(f"  ✗ {host}: Failed - {result.get('error', 'Unknown error')}")
        
        # DNS tests
        report.append("\nDNS Resolution Tests:")
        for key, result in self.results.items():
            if key.startswith('dns_'):
                hostname = key.replace('dns_', '')
                if result['success']:
                    report.append(f"  ✓ {hostname} → {result['ip']}")
                else:
                    report.append(f"  ✗ {hostname}: {result.get('error', 'Failed')}")
        
        # Port tests
        report.append("\nPort Connectivity Tests:")
        for key, result in self.results.items():
            if key.startswith('port_'):
                parts = key.replace('port_', '').rsplit('_', 1)
                host, port = parts[0], parts[1]
                status = "✓ Open" if result['success'] else "✗ Closed/Filtered"
                report.append(f"  {host}:{port} - {status}")
        
        return "\n".join(report)

# Usage
if __name__ == "__main__":
    troubleshooter = NetworkTroubleshooter()
    results = troubleshooter.comprehensive_test()
    print(troubleshooter.generate_report())
```

---

## DNS and DHCP

### Domain Name System (DNS)
```bash
# DNS Record Types
A       # IPv4 address
AAAA    # IPv6 address
CNAME   # Canonical name (alias)
MX      # Mail exchange
NS      # Name server
PTR     # Pointer (reverse DNS)
SOA     # Start of authority
TXT     # Text record
SRV     # Service record

# DNS Query Process
1. Client queries local DNS cache
2. If not cached, query local DNS resolver
3. Resolver queries root DNS servers
4. Root servers refer to TLD servers
5. TLD servers refer to authoritative servers
6. Authoritative servers return the answer
7. Answer is cached and returned to client
```

### DNS Troubleshooting
```bash
# DNS Testing Commands
nslookup google.com             # Basic DNS lookup
nslookup google.com 8.8.8.8     # Query specific DNS server

dig google.com                  # Detailed DNS query
dig @8.8.8.8 google.com         # Query specific server
dig google.com MX               # Query specific record type
dig +trace google.com           # Trace DNS resolution path

host google.com                 # Simple DNS lookup
host -t MX google.com           # Query specific record type

# Reverse DNS lookup
dig -x 8.8.8.8                  # Reverse lookup
nslookup 8.8.8.8                # Reverse lookup

# DNS Cache
sudo systemctl flush-dns        # Flush DNS cache (systemd-resolved)
sudo dscacheutil -flushcache    # Flush DNS cache (macOS)
```

### DHCP (Dynamic Host Configuration Protocol)
```bash
# DHCP Process (DORA)
1. Discover  - Client broadcasts DHCP discover
2. Offer     - Server offers IP configuration
3. Request   - Client requests offered configuration
4. Acknowledge - Server acknowledges and assigns IP

# DHCP Options
Option 1:   Subnet Mask
Option 3:   Default Gateway
Option 6:   DNS Servers
Option 12:  Hostname
Option 15:  Domain Name
Option 51:  Lease Time
Option 66:  TFTP Server
Option 67:  Boot Filename
```

### DHCP Troubleshooting
```bash
# DHCP Client Commands
sudo dhclient -r                # Release DHCP lease
sudo dhclient                   # Renew DHCP lease
sudo dhclient -v eth0           # Verbose DHCP renewal

# Check DHCP lease information
cat /var/lib/dhcp/dhclient.leases
cat /var/lib/dhclient/dhclient.leases

# DHCP server logs
tail -f /var/log/syslog | grep dhcp
journalctl -u isc-dhcp-server -f
```

---

## Wireless Networking

### WiFi Standards
```bash
# IEEE 802.11 Standards
802.11a     # 5 GHz, 54 Mbps
802.11b     # 2.4 GHz, 11 Mbps
802.11g     # 2.4 GHz, 54 Mbps
802.11n     # 2.4/5 GHz, 600 Mbps (WiFi 4)
802.11ac    # 5 GHz, 6.93 Gbps (WiFi 5)
802.11ax    # 2.4/5 GHz, 9.6 Gbps (WiFi 6)
802.11be    # 2.4/5/6 GHz, 46 Gbps (WiFi 7)
```

### WiFi Security
```bash
# Security Protocols
WEP         # Wired Equivalent Privacy (deprecated)
WPA         # WiFi Protected Access
WPA2        # WPA version 2 (AES encryption)
WPA3        # Latest WPA version (enhanced security)

# Authentication Methods
PSK         # Pre-Shared Key
EAP         # Extensible Authentication Protocol
802.1X      # Port-based authentication
```

### WiFi Troubleshooting
```bash
# WiFi Interface Commands
iwconfig                        # Show wireless interfaces
iw dev                          # Show wireless devices
iw dev wlan0 scan               # Scan for networks
iw dev wlan0 link               # Show connection info

# WiFi Signal Analysis
iwlist wlan0 scan               # Detailed scan results
wavemon                         # Real-time WiFi monitoring
iperf3 -c server_ip             # Bandwidth testing

# WiFi Connection
nmcli dev wifi list             # List available networks
nmcli dev wifi connect SSID password PASSWORD
nmcli con show                  # Show connections
nmcli con up connection_name    # Connect to network

# WiFi Diagnostics
dmesg | grep -i wifi            # Check kernel messages
journalctl -u NetworkManager    # NetworkManager logs
cat /proc/net/wireless          # Wireless statistics
```

### WiFi Performance Analysis
```python
import subprocess
import re
import json
from datetime import datetime

class WiFiAnalyzer:
    def __init__(self, interface='wlan0'):
        self.interface = interface
        self.results = {}
    
    def get_connection_info(self):
        """Get current WiFi connection information"""
        try:
            # Get connection details
            result = subprocess.run(
                ['iw', 'dev', self.interface, 'link'],
                capture_output=True,
                text=True
            )
            
            if result.returncode == 0:
                output = result.stdout
                info = {}
                
                # Parse connection info
                if 'Connected to' in output:
                    bssid_match = re.search(r'Connected to ([a-fA-F0-9:]{17})', output)
                    if bssid_match:
                        info['bssid'] = bssid_match.group(1)
                    
                    ssid_match = re.search(r'SSID: (.+)', output)
                    if ssid_match:
                        info['ssid'] = ssid_match.group(1)
                    
                    freq_match = re.search(r'freq: (\d+)', output)
                    if freq_match:
                        info['frequency'] = int(freq_match.group(1))
                        info['band'] = '5GHz' if int(freq_match.group(1)) > 3000 else '2.4GHz'
                    
                    signal_match = re.search(r'signal: (-?\d+) dBm', output)
                    if signal_match:
                        info['signal_strength'] = int(signal_match.group(1))
                        info['signal_quality'] = self.dbm_to_quality(int(signal_match.group(1)))
                
                self.results['connection_info'] = info
                return info
            
        except Exception as e:
            self.results['connection_error'] = str(e)
        
        return None
    
    def dbm_to_quality(self, dbm):
        """Convert dBm to signal quality percentage"""
        if dbm >= -30:
            return 100
        elif dbm >= -67:
            return 2 * (dbm + 100)
        elif dbm >= -70:
            return 2 * (dbm + 100) - 1
        elif dbm >= -80:
            return 2 * (dbm + 100) - 2
        elif dbm >= -90:
            return 2 * (dbm + 100) - 3
        else:
            return 0
    
    def scan_networks(self):
        """Scan for available WiFi networks"""
        try:
            result = subprocess.run(
                ['iw', 'dev', self.interface, 'scan'],
                capture_output=True,
                text=True
            )
            
            if result.returncode == 0:
                networks = []
                current_network = {}
                
                for line in result.stdout.split('\n'):
                    line = line.strip()
                    
                    if line.startswith('BSS '):
                        if current_network:
                            networks.append(current_network)
                        current_network = {'bssid': line.split()[1].rstrip(':')}
                    
                    elif 'SSID:' in line:
                        ssid = line.split('SSID: ', 1)[1]
                        current_network['ssid'] = ssid
                    
                    elif 'signal:' in line:
                        signal_match = re.search(r'signal: (-?\d+\.\d+) dBm', line)
                        if signal_match:
                            dbm = float(signal_match.group(1))
                            current_network['signal_strength'] = dbm
                            current_network['signal_quality'] = self.dbm_to_quality(dbm)
                    
                    elif 'freq:' in line:
                        freq_match = re.search(r'freq: (\d+)', line)
                        if freq_match:
                            freq = int(freq_match.group(1))
                            current_network['frequency'] = freq
                            current_network['band'] = '5GHz' if freq > 3000 else '2.4GHz'
                
                if current_network:
                    networks.append(current_network)
                
                # Sort by signal strength
                networks.sort(key=lambda x: x.get('signal_strength', -100), reverse=True)
                self.results['available_networks'] = networks
                return networks
            
        except Exception as e:
            self.results['scan_error'] = str(e)
        
        return []
    
    def get_statistics(self):
        """Get WiFi interface statistics"""
        try:
            # Get interface statistics
            with open(f'/proc/net/dev', 'r') as f:
                for line in f:
                    if self.interface in line:
                        parts = line.split()
                        stats = {
                            'rx_bytes': int(parts[1]),
                            'rx_packets': int(parts[2]),
                            'rx_errors': int(parts[3]),
                            'rx_dropped': int(parts[4]),
                            'tx_bytes': int(parts[9]),
                            'tx_packets': int(parts[10]),
                            'tx_errors': int(parts[11]),
                            'tx_dropped': int(parts[12])
                        }
                        self.results['interface_stats'] = stats
                        return stats
            
            # Get wireless statistics
            with open('/proc/net/wireless', 'r') as f:
                for line in f:
                    if self.interface in line:
                        parts = line.split()
                        wireless_stats = {
                            'link_quality': parts[2].rstrip('.'),
                            'signal_level': parts[3].rstrip('.'),
                            'noise_level': parts[4].rstrip('.')
                        }
                        self.results['wireless_stats'] = wireless_stats
                        return wireless_stats
        
        except Exception as e:
            self.results['stats_error'] = str(e)
        
        return None
    
    def test_bandwidth(self, server_ip, duration=10):
        """Test WiFi bandwidth using iperf3"""
        try:
            # Test download speed
            result = subprocess.run(
                ['iperf3', '-c', server_ip, '-t', str(duration), '-J'],
                capture_output=True,
                text=True,
                timeout=duration + 10
            )
            
            if result.returncode == 0:
                data = json.loads(result.stdout)
                bandwidth_info = {
                    'download_mbps': data['end']['sum_received']['bits_per_second'] / 1000000,
                    'upload_mbps': None,  # Would need separate test
                    'retransmits': data['end']['sum_sent']['retransmits'],
                    'jitter_ms': data['end']['sum_received']['jitter_ms']
                }
                self.results['bandwidth_test'] = bandwidth_info
                return bandwidth_info
        
        except Exception as e:
            self.results['bandwidth_error'] = str(e)
        
        return None
    
    def generate_report(self):
        """Generate WiFi analysis report"""
        report = []
        report.append("WiFi Analysis Report")
        report.append("=" * 30)
        report.append(f"Interface: {self.interface}")
        report.append(f"Generated: {datetime.now()}")
        report.append("")
        
        # Connection info
        if 'connection_info' in self.results:
            info = self.results['connection_info']
            report.append("Current Connection:")
            report.append(f"  SSID: {info.get('ssid', 'N/A')}")
            report.append(f"  BSSID: {info.get('bssid', 'N/A')}")
            report.append(f"  Frequency: {info.get('frequency', 'N/A')} MHz ({info.get('band', 'N/A')})")
            report.append(f"  Signal Strength: {info.get('signal_strength', 'N/A')} dBm")
            report.append(f"  Signal Quality: {info.get('signal_quality', 'N/A')}%")
            report.append("")
        
        # Available networks
        if 'available_networks' in self.results:
            networks = self.results['available_networks'][:10]  # Top 10
            report.append("Available Networks (Top 10):")
            for net in networks:
                ssid = net.get('ssid', 'Hidden')
                signal = net.get('signal_strength', 'N/A')
                band = net.get('band', 'N/A')
                report.append(f"  {ssid:<20} {signal:>6} dBm ({band})")
            report.append("")
        
        # Statistics
        if 'interface_stats' in self.results:
            stats = self.results['interface_stats']
            report.append("Interface Statistics:")
            report.append(f"  RX: {stats['rx_bytes']:,} bytes, {stats['rx_packets']:,} packets")
            report.append(f"  TX: {stats['tx_bytes']:,} bytes, {stats['tx_packets']:,} packets")
            report.append(f"  Errors: RX={stats['rx_errors']}, TX={stats['tx_errors']}")
            report.append(f"  Dropped: RX={stats['rx_dropped']}, TX={stats['tx_dropped']}")
            report.append("")
        
        # Bandwidth test
        if 'bandwidth_test' in self.results:
            bw = self.results['bandwidth_test']
            report.append("Bandwidth Test:")
            report.append(f"  Download: {bw['download_mbps']:.2f} Mbps")
            report.append(f"  Retransmits: {bw['retransmits']}")
            report.append(f"  Jitter: {bw['jitter_ms']:.2f} ms")
        
        return "\n".join(report)

# Usage
if __name__ == "__main__":
    analyzer = WiFiAnalyzer()
    analyzer.get_connection_info()
    analyzer.scan_networks()
    analyzer.get_statistics()
    print(analyzer.generate_report())
```

---

## Network Security

### Firewall Basics
```bash
# iptables (Linux firewall)
# Basic iptables commands
iptables -L                     # List all rules
iptables -L -n                  # List rules with numeric output
iptables -F                     # Flush all rules
iptables -P INPUT DROP          # Set default policy to DROP

# Allow specific traffic
iptables -A INPUT -p tcp --dport 22 -j ACCEPT      # Allow SSH
iptables -A INPUT -p tcp --dport 80 -j ACCEPT      # Allow HTTP
iptables -A INPUT -p tcp --dport 443 -j ACCEPT     # Allow HTTPS
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Block specific IP
iptables -A INPUT -s 192.168.1.100 -j DROP

# Save rules
iptables-save > /etc/iptables/rules.v4
```

### Network Security Monitoring
```python
import subprocess
import re
import time
from datetime import datetime
from collections import defaultdict

class NetworkSecurityMonitor:
    def __init__(self):
        self.suspicious_ips = set()
        self.connection_counts = defaultdict(int)
        self.failed_attempts = defaultdict(int)
    
    def monitor_connections(self):
        """Monitor active network connections"""
        try:
            result = subprocess.run(
                ['ss', '-tuln'],
                capture_output=True,
                text=True
            )
            
            connections = []
            for line in result.stdout.split('\n')[1:]:  # Skip header
                if line.strip():
                    parts = line.split()
                    if len(parts) >= 5:
                        connections.append({
                            'protocol': parts[0],
                            'state': parts[1],
                            'local_address': parts[4],
                            'remote_address': parts[5] if len(parts) > 5 else 'N/A'
                        })
            
            return connections
        except Exception as e:
            print(f"Error monitoring connections: {e}")
            return []
    
    def check_failed_logins(self):
        """Check for failed login attempts"""
        try:
            result = subprocess.run(
                ['grep', 'Failed password', '/var/log/auth.log'],
                capture_output=True,
                text=True
            )
            
            failed_logins = []
            for line in result.stdout.split('\n'):
                if line.strip():
                    # Extract IP address
                    ip_match = re.search(r'from (\d+\.\d+\.\d+\.\d+)', line)
                    if ip_match:
                        ip = ip_match.group(1)
                        self.failed_attempts[ip] += 1
                        
                        # Mark as suspicious if too many attempts
                        if self.failed_attempts[ip] > 5:
                            self.suspicious_ips.add(ip)
                        
                        failed_logins.append({
                            'ip': ip,
                            'line': line.strip(),
                            'count': self.failed_attempts[ip]
                        })
            
            return failed_logins
        except Exception as e:
            print(f"Error checking failed logins: {e}")
            return []
    
    def scan_open_ports(self, target='localhost'):
        """Scan for open ports"""
        try:
            result = subprocess.run(
                ['nmap', '-sT', '-O', target],
                capture_output=True,
                text=True,
                timeout=60
            )
            
            open_ports = []
            for line in result.stdout.split('\n'):
                if '/tcp' in line and 'open' in line:
                    parts = line.split()
                    port = parts[0].split('/')[0]
                    service = parts[2] if len(parts) > 2 else 'unknown'
                    open_ports.append({
                        'port': port,
                        'service': service
                    })
            
            return open_ports
        except Exception as e:
            print(f"Error scanning ports: {e}")
            return []
    
    def generate_security_report(self):
        """Generate security monitoring report"""
        report = []
        report.append("Network Security Report")
        report.append("=" * 30)
        report.append(f"Generated: {datetime.now()}")
        report.append("")
        
        # Active connections
        connections = self.monitor_connections()
        report.append(f"Active Connections: {len(connections)}")
        
        # Failed login attempts
        failed_logins = self.check_failed_logins()
        if failed_logins:
            report.append(f"\nFailed Login Attempts: {len(failed_logins)}")
            for attempt in failed_logins[-10:]:  # Last 10
                report.append(f"  {attempt['ip']} ({attempt['count']} attempts)")
        
        # Suspicious IPs
        if self.suspicious_ips:
            report.append(f"\nSuspicious IPs: {len(self.suspicious_ips)}")
            for ip in self.suspicious_ips:
                report.append(f"  {ip}")
        
        # Open ports
        open_ports = self.scan_open_ports()
        if open_ports:
            report.append(f"\nOpen Ports:")
            for port_info in open_ports:
                report.append(f"  {port_info['port']}: {port_info['service']}")
        
        return "\n".join(report)
```

---

## Network Monitoring

### Bandwidth Monitoring
```bash
# Monitor bandwidth usage
iftop                           # Real-time bandwidth by connection
iftop -i eth0                   # Monitor specific interface
nethogs                        # Bandwidth by process
vnstat                         # Network statistics
vnstat -i eth0 -h              # Hourly stats for interface

# Monitor network traffic
tcpdump -i eth0                 # Capture packets
tcpdump -i eth0 port 80         # Capture HTTP traffic
tcpdump -i eth0 host 8.8.8.8    # Capture traffic to/from specific host

# Network statistics
cat /proc/net/dev               # Interface statistics
ss -s                          # Socket statistics summary
netstat -i                     # Interface statistics
```

### Network Performance Testing
```python
import subprocess
import time
import statistics
from datetime import datetime

class NetworkPerformanceTester:
    def __init__(self):
        self.results = {}
    
    def ping_test(self, host, count=10):
        """Perform ping test and calculate statistics"""
        try:
            result = subprocess.run(
                ['ping', '-c', str(count), host],
                capture_output=True,
                text=True
            )
            
            if result.returncode == 0:
                # Parse ping results
                lines = result.stdout.split('\n')
                times = []
                
                for line in lines:
                    if 'time=' in line:
                        time_match = re.search(r'time=(\d+\.?\d*)', line)
                        if time_match:
                            times.append(float(time_match.group(1)))
                
                if times:
                    stats = {
                        'host': host,
                        'count': len(times),
                        'min': min(times),
                        'max': max(times),
                        'avg': statistics.mean(times),
                        'median': statistics.median(times),
                        'stdev': statistics.stdev(times) if len(times) > 1 else 0
                    }
                    
                    # Calculate packet loss
                    for line in lines:
                        if 'packet loss' in line:
                            loss_match = re.search(r'(\d+)% packet loss', line)
                            if loss_match:
                                stats['packet_loss'] = int(loss_match.group(1))
                    
                    self.results[f'ping_{host}'] = stats
                    return stats
        
        except Exception as e:
            self.results[f'ping_{host}_error'] = str(e)
        
        return None
    
    def bandwidth_test(self, server, duration=10):
        """Test bandwidth using iperf3"""
        try:
            # Test download
            result = subprocess.run(
                ['iperf3', '-c', server, '-t', str(duration), '-J'],
                capture_output=True,
                text=True,
                timeout=duration + 10
            )
            
            if result.returncode == 0:
                import json
                data = json.loads(result.stdout)
                
                download_bps = data['end']['sum_received']['bits_per_second']
                upload_bps = data['end']['sum_sent']['bits_per_second']
                
                bandwidth_stats = {
                    'server': server,
                    'download_mbps': download_bps / 1000000,
                    'upload_mbps': upload_bps / 1000000,
                    'retransmits': data['end']['sum_sent']['retransmits'],
                    'jitter_ms': data['end']['sum_received']['jitter_ms']
                }
                
                self.results[f'bandwidth_{server}'] = bandwidth_stats
                return bandwidth_stats
        
        except Exception as e:
            self.results[f'bandwidth_{server}_error'] = str(e)
        
        return None
    
    def traceroute_analysis(self, host):
        """Analyze network path to host"""
        try:
            result = subprocess.run(
                ['traceroute', host],
                capture_output=True,
                text=True,
                timeout=60
            )
            
            if result.returncode == 0:
                hops = []
                lines = result.stdout.split('\n')
                
                for line in lines:
                    if line.strip() and not line.startswith('traceroute'):
                        hop_match = re.match(r'\s*(\d+)\s+(.+)', line)
                        if hop_match:
                            hop_num = int(hop_match.group(1))
                            hop_data = hop_match.group(2)
                            
                            # Extract IP and timing
                            ip_match = re.search(r'(\d+\.\d+\.\d+\.\d+)', hop_data)
                            time_matches = re.findall(r'(\d+\.?\d*) ms', hop_data)
                            
                            hop_info = {
                                'hop': hop_num,
                                'ip': ip_match.group(1) if ip_match else 'N/A',
                                'times': [float(t) for t in time_matches]
                            }
                            
                            if hop_info['times']:
                                hop_info['avg_time'] = statistics.mean(hop_info['times'])
                            
                            hops.append(hop_info)
                
                traceroute_stats = {
                    'host': host,
                    'total_hops': len(hops),
                    'hops': hops
                }
                
                self.results[f'traceroute_{host}'] = traceroute_stats
                return traceroute_stats
        
        except Exception as e:
            self.results[f'traceroute_{host}_error'] = str(e)
        
        return None
    
    def comprehensive_test(self, targets=None):
        """Run comprehensive network performance tests"""
        if targets is None:
            targets = ['8.8.8.8', 'google.com', '1.1.1.1']
        
        print("Starting network performance tests...")
        
        for target in targets:
            print(f"Testing {target}...")
            
            # Ping test
            self.ping_test(target)
            
            # Traceroute analysis
            self.traceroute_analysis(target)
        
        return self.results
    
    def generate_performance_report(self):
        """Generate performance test report"""
        report = []
        report.append("Network Performance Report")
        report.append("=" * 35)
        report.append(f"Generated: {datetime.now()}")
        report.append("")
        
        # Ping results
        for key, result in self.results.items():
            if key.startswith('ping_') and not key.endswith('_error'):
                host = result['host']
                report.append(f"Ping Test - {host}:")
                report.append(f"  Packets: {result['count']}, Loss: {result.get('packet_loss', 0)}%")
                report.append(f"  Latency: min={result['min']:.1f}ms, avg={result['avg']:.1f}ms, max={result['max']:.1f}ms")
                report.append(f"  Std Dev: {result['stdev']:.1f}ms")
                report.append("")
        
        # Traceroute results
        for key, result in self.results.items():
            if key.startswith('traceroute_') and not key.endswith('_error'):
                host = result['host']
                report.append(f"Traceroute - {host}:")
                report.append(f"  Total Hops: {result['total_hops']}")
                
                # Show first few and last few hops
                hops_to_show = result['hops'][:3] + result['hops'][-3:]
                for hop in hops_to_show:
                    avg_time = hop.get('avg_time', 'N/A')
                    report.append(f"  Hop {hop['hop']}: {hop['ip']} ({avg_time}ms)")
                report.append("")
        
        return "\n".join(report)
```

---

## Device Connectivity

### Amazon Device Network Requirements
```bash
# Fire TV Network Requirements
- Bandwidth: Minimum 1 Mbps for SD, 5 Mbps for HD, 25 Mbps for 4K
- Latency: < 100ms for optimal streaming
- Ports: 80 (HTTP), 443 (HTTPS), 53 (DNS)
- Protocols: TCP, UDP, DHCP

# Echo Device Network Requirements
- Bandwidth: Minimum 512 Kbps
- Latency: < 200ms for voice response
- Ports: 80, 443, 4070 (voice), 40317-40327 (media)
- Protocols: TCP, UDP, HTTP/2

# Kindle Network Requirements
- Bandwidth: Minimum 256 Kbps
- Ports: 80, 443
- Protocols: TCP, HTTP, HTTPS
```

### Device Network Troubleshooting
```python
class DeviceNetworkTroubleshooter:
    def __init__(self, device_type='fire_tv'):
        self.device_type = device_type
        self.requirements = self.get_device_requirements()
        self.results = {}
    
    def get_device_requirements(self):
        """Get network requirements for device type"""
        requirements = {
            'fire_tv': {
                'min_bandwidth_mbps': 5,
                'max_latency_ms': 100,
                'required_ports': [80, 443, 53],
                'streaming_servers': ['netflix.com', 'primevideo.com', 'hulu.com']
            },
            'echo': {
                'min_bandwidth_mbps': 0.5,
                'max_latency_ms': 200,
                'required_ports': [80, 443, 4070],
                'voice_servers': ['alexa.amazon.com']
            },
            'kindle': {
                'min_bandwidth_mbps': 0.25,
                'max_latency_ms': 300,
                'required_ports': [80, 443],
                'content_servers': ['kindle.amazon.com']
            }
        }
        return requirements.get(self.device_type, requirements['fire_tv'])
    
    def test_bandwidth_requirements(self):
        """Test if bandwidth meets device requirements"""
        # This would typically use iperf3 or similar
        # For demo, we'll simulate the test
        try:
            # Simulate bandwidth test
            measured_bandwidth = 10.5  # Mbps
            
            meets_requirement = measured_bandwidth >= self.requirements['min_bandwidth_mbps']
            
            result = {
                'measured_mbps': measured_bandwidth,
                'required_mbps': self.requirements['min_bandwidth_mbps'],
                'meets_requirement': meets_requirement,
                'margin': measured_bandwidth - self.requirements['min_bandwidth_mbps']
            }
            
            self.results['bandwidth_test'] = result
            return result
        
        except Exception as e:
            self.results['bandwidth_error'] = str(e)
            return None
    
    def test_latency_requirements(self):
        """Test if latency meets device requirements"""
        try:
            # Test latency to key servers
            servers = self.requirements.get('streaming_servers', ['amazon.com'])
            latency_results = {}
            
            for server in servers:
                result = subprocess.run(
                    ['ping', '-c', '5', server],
                    capture_output=True,
                    text=True
                )
                
                if result.returncode == 0:
                    # Parse average latency
                    for line in result.stdout.split('\n'):
                        if 'min/avg/max' in line:
                            avg_latency = float(line.split('/')[1])
                            meets_requirement = avg_latency <= self.requirements['max_latency_ms']
                            
                            latency_results[server] = {
                                'measured_ms': avg_latency,
                                'required_ms': self.requirements['max_latency_ms'],
                                'meets_requirement': meets_requirement
                            }
            
            self.results['latency_test'] = latency_results
            return latency_results
        
        except Exception as e:
            self.results['latency_error'] = str(e)
            return None
    
    def test_port_connectivity(self):
        """Test connectivity to required ports"""
        port_results = {}
        
        for port in self.requirements['required_ports']:
            try:
                sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                sock.settimeout(5)
                result = sock.connect_ex(('amazon.com', port))
                sock.close()
                
                port_results[port] = {
                    'accessible': result == 0,
                    'port': port
                }
            
            except Exception as e:
                port_results[port] = {
                    'accessible': False,
                    'error': str(e)
                }
        
        self.results['port_test'] = port_results
        return port_results
    
    def diagnose_device_connectivity(self):
        """Run comprehensive device connectivity diagnosis"""
        print(f"Diagnosing {self.device_type} connectivity...")
        
        # Test bandwidth
        self.test_bandwidth_requirements()
        
        # Test latency
        self.test_latency_requirements()
        
        # Test port connectivity
        self.test_port_connectivity()
        
        return self.results
    
    def generate_device_report(self):
        """Generate device-specific connectivity report"""
        report = []
        report.append(f"{self.device_type.upper()} Connectivity Report")
        report.append("=" * 40)
        report.append(f"Generated: {datetime.now()}")
        report.append("")
        
        # Bandwidth results
        if 'bandwidth_test' in self.results:
            bw = self.results['bandwidth_test']
            status = "✓ PASS" if bw['meets_requirement'] else "✗ FAIL"
            report.append(f"Bandwidth Test: {status}")
            report.append(f"  Measured: {bw['measured_mbps']} Mbps")
            report.append(f"  Required: {bw['required_mbps']} Mbps")
            report.append(f"  Margin: {bw['margin']:.1f} Mbps")
            report.append("")
        
        # Latency results
        if 'latency_test' in self.results:
            report.append("Latency Tests:")
            for server, result in self.results['latency_test'].items():
                status = "✓ PASS" if result['meets_requirement'] else "✗ FAIL"
                report.append(f"  {server}: {status}")
                report.append(f"    Measured: {result['measured_ms']:.1f} ms")
                report.append(f"    Required: < {result['required_ms']} ms")
            report.append("")
        
        # Port connectivity results
        if 'port_test' in self.results:
            report.append("Port Connectivity:")
            for port, result in self.results['port_test'].items():
                status = "✓ OPEN" if result['accessible'] else "✗ BLOCKED"
                report.append(f"  Port {port}: {status}")
            report.append("")
        
        return "\n".join(report)
```

---

## Performance Analysis

### Network Performance Metrics
```bash
# Key Network Performance Indicators
Bandwidth       # Data transfer rate (Mbps/Gbps)
Latency         # Round-trip time (ms)
Jitter          # Latency variation (ms)
Packet Loss     # Percentage of lost packets
Throughput      # Actual data transfer rate
Utilization     # Percentage of bandwidth used
```

### Performance Monitoring Tools
```bash
# Bandwidth monitoring
iftop -i eth0                   # Real-time bandwidth by connection
nload eth0                      # Real-time bandwidth graph
bmon                           # Bandwidth monitor
cbm                            # Color bandwidth meter

# Latency monitoring
ping -i 0.2 8.8.8.8            # High-frequency ping
mtr --report-cycles 100 google.com  # MTR report mode
hping3 -S -p 80 google.com     # Advanced ping tool

# Packet analysis
tcpdump -i eth0 -c 100         # Capture 100 packets
wireshark                      # GUI packet analyzer
tshark -i eth0 -c 100          # Command-line Wireshark
```

---

## Common Issues and Solutions

### Connectivity Issues
```bash
# Issue: No Internet Connection
Troubleshooting Steps:
1. Check physical connections
2. Verify IP configuration: ip addr show
3. Test local gateway: ping gateway_ip
4. Test DNS resolution: nslookup google.com
5. Test external connectivity: ping 8.8.8.8
6. Check routing table: ip route show

# Issue: Slow Network Performance
Troubleshooting Steps:
1. Test bandwidth: iperf3 -c server
2. Check for packet loss: ping -c 100 target
3. Analyze network path: mtr target
4. Monitor interface errors: ip -s link show
5. Check for duplex mismatches
6. Verify QoS settings
```

### DNS Issues
```bash
# Issue: DNS Resolution Failures
Troubleshooting Steps:
1. Test with different DNS servers:
   nslookup google.com 8.8.8.8
   nslookup google.com 1.1.1.1
2. Check DNS configuration: cat /etc/resolv.conf
3. Flush DNS cache: sudo systemctl flush-dns
4. Test reverse DNS: nslookup ip_address
5. Check for DNS filtering/blocking
```

### WiFi Issues
```bash
# Issue: WiFi Connection Problems
Troubleshooting Steps:
1. Check signal strength: iwconfig wlan0
2. Scan for networks: iw dev wlan0 scan
3. Check for interference: iwlist wlan0 scan
4. Verify authentication: check WPA/WPA2 settings
5. Test different channels
6. Check power management: iwconfig wlan0 power off
```

---

## Network Tools and Commands

### Essential Network Commands Reference
```bash
# Interface Management
ip addr show                    # Show IP addresses
ip link set eth0 up            # Bring interface up
ip link set eth0 down          # Bring interface down
ip addr add 192.168.1.100/24 dev eth0  # Add IP address

# Routing
ip route show                   # Show routing table
ip route add default via 192.168.1.1   # Add default route
ip route add 10.0.0.0/8 via 192.168.1.1  # Add specific route
ip route del 10.0.0.0/8        # Delete route

# ARP
arp -a                         # Show ARP table
ip neigh show                  # Show neighbor table
arp -d 192.168.1.1            # Delete ARP entry

# Network Statistics
ss -tuln                       # Show listening sockets
ss -tulpn                      # Show sockets with processes
netstat -rn                    # Show routing table
netstat -i                     # Show interface statistics

# Packet Capture
tcpdump -i eth0                # Capture on interface
tcpdump -i eth0 port 80        # Capture HTTP traffic
tcpdump -i eth0 host 8.8.8.8   # Capture traffic to/from host
tcpdump -w capture.pcap        # Write to file
```

---

## Interview Scenarios

### Scenario 1: Fire TV Streaming Issues
**Question**: "A customer reports that their Fire TV is buffering frequently during Netflix streaming. How would you troubleshoot this network issue?"

**Answer Approach**:
```bash
# 1. Check bandwidth requirements
# Netflix requires: SD=3Mbps, HD=5Mbps, 4K=25Mbps
iperf3 -c netflix_server -t 30

# 2. Test connectivity to Netflix servers
ping netflix.com
traceroute netflix.com
nslookup netflix.com

# 3. Check for packet loss and latency
ping -c 100 netflix.com
mtr --report-cycles 50 netflix.com

# 4. Analyze WiFi connection (if applicable)
iwconfig wlan0
iw dev wlan0 scan | grep -A5 -B5 "current_network"

# 5. Check for network congestion
iftop -i wlan0
nethogs

# 6. Test different times of day
# Peak usage hours may show congestion
```

### Scenario 2: Echo Device Not Responding
**Question**: "An Echo device is not responding to voice commands. How would you diagnose the network connectivity?"

**Answer Approach**:
```bash
# 1. Check basic connectivity
ping 8.8.8.8
ping alexa.amazon.com

# 2. Test required ports for Alexa
nc -zv alexa.amazon.com 443
nc -zv alexa.amazon.com 4070

# 3. Check DNS resolution
nslookup alexa.amazon.com
dig alexa.amazon.com

# 4. Test latency (critical for voice response)
ping -c 20 alexa.amazon.com
# Should be < 200ms for optimal performance

# 5. Check for firewall blocking
iptables -L | grep -i drop
# Ensure ports 80, 443, 4070 are not blocked

# 6. Verify time synchronization
ntpdate -q pool.ntp.org
# Echo devices require accurate time
```

### Scenario 3: Multiple Device Connectivity Issues
**Question**: "Multiple Amazon devices in a home are experiencing connectivity issues. How would you approach this systematically?"

**Answer Approach**:
```bash
# 1. Check ISP/WAN connectivity
ping 8.8.8.8
traceroute 8.8.8.8
# Test from router/gateway

# 2. Check local network infrastructure
ping gateway_ip
arp -a  # Check for duplicate IPs
ip route show  # Verify routing

# 3. Check DHCP server
cat /var/lib/dhcp/dhclient.leases
# Look for IP conflicts or lease issues

# 4. Test DNS resolution
nslookup amazon.com 8.8.8.8
nslookup amazon.com local_dns_server

# 5. Check bandwidth utilization
iftop -i eth0
# Look for bandwidth-heavy applications

# 6. Analyze WiFi environment (if applicable)
iwlist wlan0 scan
# Check for interference, channel congestion

# 7. Test device-specific requirements
# Fire TV: 5+ Mbps, <100ms latency
# Echo: 0.5+ Mbps, <200ms latency
# Kindle: 0.25+ Mbps, <300ms latency
```

### Key Interview Points to Remember:
1. **Systematic Approach**: Use OSI model layers for troubleshooting
2. **Device-Specific Knowledge**: Understand requirements for different Amazon devices
3. **Tool Proficiency**: Demonstrate knowledge of network diagnostic tools
4. **Performance Analysis**: Know how to measure and interpret network metrics
5. **Security Awareness**: Understand network security implications
6. **Documentation**: Always document findings and solutions
7. **Escalation**: Know when to escalate to network specialists or ISP
