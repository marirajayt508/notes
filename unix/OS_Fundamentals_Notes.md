# Operating System Fundamentals - Amazon Lab126 Support Engineer

## Table of Contents
1. [Process Management](#process-management)
2. [Memory Management](#memory-management)
3. [File Systems](#file-systems)
4. [Inter-Process Communication](#inter-process-communication)
5. [System Calls](#system-calls)
6. [I/O Operations](#io-operations)
7. [Device OS Concepts](#device-os-concepts)
8. [Interview Focus Areas](#interview-focus-areas)

---

## Process Management

### Core Concepts
- **Process**: Running instance of a program with its own memory space
- **Thread**: Lightweight process sharing memory space with parent process
- **Daemon**: Background process that runs continuously

### Process States
```
NEW → READY → RUNNING → WAITING → TERMINATED
                ↑         ↓
                ←---------←
```

- **Running**: Currently executing on CPU
- **Ready**: Waiting for CPU time
- **Waiting/Blocked**: Waiting for I/O or resource
- **Zombie**: Terminated but parent hasn't read exit status
- **Stopped**: Suspended by signal (SIGSTOP)

### Process Scheduling
- **Round Robin**: Each process gets equal CPU time slice
- **Priority Scheduling**: Higher priority processes run first
- **Shortest Job First**: Minimize average waiting time
- **Multilevel Queue**: Different queues for different process types

### Key System Calls
```c
pid_t fork();        // Create child process
int exec();          // Replace process image
int wait();          // Wait for child process
int kill(pid, sig);  // Send signal to process
```

### Commands to Master
```bash
# Process viewing
ps aux                    # All processes with details
ps -eo pid,ppid,cmd      # Custom format
pstree                   # Process tree
top                      # Real-time process monitor
htop                     # Enhanced top

# Process control
kill PID                 # Terminate process (SIGTERM)
kill -9 PID             # Force kill (SIGKILL)
killall process_name    # Kill by name
pkill -f pattern        # Kill by pattern
nohup command &         # Run in background, ignore hangup

# Process information
/proc/PID/status        # Process status
/proc/PID/cmdline       # Command line
/proc/PID/environ       # Environment variables
/proc/PID/maps          # Memory mappings
```

---

## Memory Management

### Virtual Memory Concepts
- **Virtual Address Space**: Each process has its own virtual memory
- **Physical Memory**: Actual RAM in the system
- **Page**: Fixed-size memory block (usually 4KB)
- **Page Table**: Maps virtual addresses to physical addresses

### Memory Segments
```
High Address
┌─────────────┐
│    Stack    │ ← Local variables, function calls
├─────────────┤
│     ↓       │
│             │
│     ↑       │
├─────────────┤
│    Heap     │ ← Dynamic allocation (malloc)
├─────────────┤
│    Data     │ ← Global/static variables
├─────────────┤
│    Text     │ ← Program code
└─────────────┘
Low Address
```

### Memory Management Techniques
- **Paging**: Divide memory into fixed-size pages
- **Swapping**: Move pages between RAM and disk
- **Memory Mapping**: Map files into memory
- **Copy-on-Write**: Share memory until modification

### Memory Monitoring Commands
```bash
# Memory usage
free -m                 # Memory usage in MB
free -h                 # Human readable format
cat /proc/meminfo       # Detailed memory info
vmstat 1 5             # Virtual memory statistics

# Process memory
pmap PID               # Process memory map
cat /proc/PID/status   # Process memory details
ps aux --sort=-%mem    # Processes by memory usage

# Memory debugging
valgrind ./program     # Memory leak detection
strace -e memory       # Trace memory system calls
```

---

## File Systems

### File System Hierarchy
```
/                    # Root directory
├── bin/            # Essential binaries
├── boot/           # Boot files
├── dev/            # Device files
├── etc/            # Configuration files
├── home/           # User directories
├── lib/            # Shared libraries
├── proc/           # Process information
├── sys/            # System information
├── tmp/            # Temporary files
├── usr/            # User programs
└── var/            # Variable data (logs)
```

### Inode Concepts
- **Inode**: Data structure storing file metadata
- **Hard Link**: Multiple directory entries pointing to same inode
- **Soft Link**: File containing path to another file
- **File Permissions**: rwx for owner, group, others

### File System Types
- **ext4**: Default Linux file system
- **xfs**: High-performance file system
- **tmpfs**: Memory-based file system
- **proc**: Virtual file system for process info
- **sysfs**: Virtual file system for system info

### File Operations Commands
```bash
# File information
ls -la                 # Detailed file listing
stat filename          # File statistics
file filename          # File type
lsof filename          # Processes using file

# File permissions
chmod 755 file         # Change permissions
chown user:group file  # Change ownership
umask 022             # Default permissions

# Links
ln file hardlink       # Create hard link
ln -s file softlink    # Create soft link

# File system
df -h                  # Disk usage
du -sh directory       # Directory size
mount                  # Show mounted file systems
lsblk                  # List block devices
```

---

## Inter-Process Communication (IPC)

### IPC Mechanisms

#### 1. Pipes
```bash
# Anonymous pipe
command1 | command2

# Named pipe (FIFO)
mkfifo mypipe
echo "data" > mypipe &
cat < mypipe
```

#### 2. Signals
```bash
# Common signals
SIGTERM (15)    # Terminate gracefully
SIGKILL (9)     # Force terminate
SIGUSR1 (10)    # User-defined signal
SIGHUP (1)      # Hangup signal

# Send signals
kill -TERM PID
kill -USR1 PID
killall -HUP process_name
```

#### 3. Shared Memory
```c
// System V shared memory
int shmget(key_t key, size_t size, int shmflg);
void *shmat(int shmid, const void *shmaddr, int shmflg);
int shmdt(const void *shmaddr);
```

#### 4. Message Queues
```bash
# View IPC resources
ipcs -q    # Message queues
ipcs -m    # Shared memory
ipcs -s    # Semaphores

# Remove IPC resources
ipcrm -q msgid
ipcrm -m shmid
```

#### 5. Sockets
```c
// Unix domain socket
int socket(AF_UNIX, SOCK_STREAM, 0);
int bind(sockfd, addr, addrlen);
int listen(sockfd, backlog);
int accept(sockfd, addr, addrlen);
```

---

## System Calls

### File Operations
```c
int open(const char *pathname, int flags);
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
int close(int fd);
int stat(const char *pathname, struct stat *statbuf);
```

### Process Operations
```c
pid_t fork(void);                    // Create child process
int execve(const char *filename, char *const argv[], char *const envp[]);
pid_t wait(int *wstatus);           // Wait for child
void exit(int status);              // Terminate process
```

### Memory Operations
```c
void *malloc(size_t size);          // Allocate memory
void free(void *ptr);               // Free memory
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
```

### System Call Debugging
```bash
# Trace system calls
strace command              # Trace all system calls
strace -e open command      # Trace specific calls
strace -p PID              # Attach to running process
strace -f command          # Follow child processes

# Library call tracing
ltrace command             # Trace library calls
ltrace -p PID             # Attach to running process
```

---

## I/O Operations

### I/O Types
- **Blocking I/O**: Process waits until operation completes
- **Non-blocking I/O**: Returns immediately, may not complete
- **Asynchronous I/O**: Operation completes in background
- **Synchronous I/O**: Process waits for completion

### File Descriptors
```
0 - stdin   (Standard input)
1 - stdout  (Standard output)
2 - stderr  (Standard error)
3+ - Other open files
```

### I/O Redirection
```bash
# Redirection
command > file              # Redirect stdout to file
command >> file             # Append stdout to file
command 2> file             # Redirect stderr to file
command &> file             # Redirect both stdout and stderr
command < file              # Redirect stdin from file

# Pipes
command1 | command2         # Pipe stdout to stdin
command1 |& command2        # Pipe both stdout and stderr
```

### I/O Monitoring
```bash
# I/O statistics
iostat -x 1 5              # Extended I/O stats
iotop                      # I/O by process
lsof                       # List open files
fuser filename             # Processes using file

# Disk I/O
sar -d 1 5                 # Disk activity
pidstat -d 1 5             # Per-process disk I/O
```

---

## Device OS Concepts

### Embedded Systems
- **Real-time OS**: Deterministic response times
- **Resource Constraints**: Limited memory, CPU, power
- **Hardware Abstraction Layer (HAL)**: Interface between OS and hardware
- **Device Drivers**: Kernel modules for hardware control

### Amazon Device OS Architecture
```
┌─────────────────────────────────────┐
│        Applications                 │
├─────────────────────────────────────┤
│        Framework Layer              │
├─────────────────────────────────────┤
│        Android Runtime              │
├─────────────────────────────────────┤
│        Linux Kernel                 │
├─────────────────────────────────────┤
│        Hardware                     │
└─────────────────────────────────────┘
```

### Key Concepts for Device Support
- **Boot Process**: Bootloader → Kernel → Init → Services
- **Power Management**: Sleep states, wake locks
- **Memory Optimization**: Limited RAM, efficient usage
- **Storage Management**: Flash memory, wear leveling
- **Network Connectivity**: WiFi, Bluetooth management
- **Update Mechanisms**: OTA updates, rollback capability

### Device Debugging
```bash
# Android Debug Bridge (for Fire devices)
adb devices                # List connected devices
adb shell                  # Access device shell
adb logcat                 # View device logs
adb pull /path/file        # Copy file from device

# System information
cat /proc/version          # Kernel version
cat /proc/cpuinfo         # CPU information
cat /proc/meminfo         # Memory information
dmesg                     # Kernel messages
```

---

## Interview Focus Areas

### Common Questions & Answers

**Q: What happens when you run a command like 'ls -la'?**
A: 
1. Shell parses command and arguments
2. fork() creates child process
3. exec() replaces child with 'ls' program
4. 'ls' makes system calls (opendir, readdir, stat)
5. Output written to stdout
6. Child process exits
7. Parent shell waits and continues

**Q: Explain the difference between process and thread**
A:
- Process: Independent memory space, expensive to create
- Thread: Shared memory space, lightweight, faster context switching
- Threads within process share heap but have separate stacks

**Q: How do you debug a process stuck in 'D' state?**
A:
1. 'D' state = Uninterruptible sleep (waiting for I/O)
2. Check what the process is waiting for: cat /proc/PID/wchan
3. Use strace to see system calls: strace -p PID
4. Check I/O statistics: iostat -x 1 5
5. Look for disk/network issues causing the block

**Q: What's the difference between hard and soft links?**
A:
- Hard link: Multiple directory entries pointing to same inode
- Soft link: File containing path to another file
- Hard links can't cross file systems or link to directories
- Deleting original file breaks soft link but not hard link

### Key Points to Remember
1. Always explain your systematic approach to problem-solving
2. Mention monitoring and prevention strategies
3. Show understanding of both theory and practical application
4. Relate concepts to device support scenarios
5. Demonstrate knowledge of debugging tools and techniques

### Practice Scenarios
1. High CPU usage investigation
2. Memory leak detection
3. Process hanging diagnosis
4. File system full troubleshooting
5. Network connectivity issues
6. Service startup failures
7. Performance degradation analysis
