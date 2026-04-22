# Appendix A: Essential Linux Commands Cheatsheet

> A practical, comprehensive reference for systems programmers. Every command includes
> syntax, the most useful flags, and real-world examples you can paste into a terminal.

---

## Table of Contents

1. [File Operations](#1-file-operations)
2. [Text Processing](#2-text-processing)
3. [Process Management](#3-process-management)
4. [System Information](#4-system-information)
5. [Networking](#5-networking)
6. [Disk and Filesystem](#6-disk-and-filesystem)
7. [Package Management](#7-package-management)
8. [User Management](#8-user-management)
9. [Archiving and Compression](#9-archiving-and-compression)
10. [systemd](#10-systemd)
11. [Performance Analysis](#11-performance-analysis)
12. [Container-Related](#12-container-related)

---

## 1. File Operations

### ls — List directory contents

```bash
ls                    # List files in current directory
ls -l                 # Long format (permissions, owner, size, date)
ls -la                # Include hidden files (dotfiles)
ls -lh                # Human-readable sizes (KB, MB, GB)
ls -ltr               # Sort by modification time, oldest first
ls -lS                # Sort by file size, largest first
ls -R                 # Recursive listing
ls -d */              # List only directories
ls -i                 # Show inode numbers
ls -1                 # One file per line (useful for scripting)
```

### cp — Copy files and directories

```bash
cp file.txt backup.txt              # Copy a file
cp -r src/ dst/                     # Copy directory recursively
cp -a src/ dst/                     # Archive mode: preserve permissions, timestamps, symlinks
cp -i file.txt dst/                 # Interactive: prompt before overwriting
cp -u src/*.c dst/                  # Copy only when source is newer
cp -v file.txt dst/                 # Verbose: show what's being copied
cp --preserve=all src dst           # Preserve all attributes (mode, ownership, timestamps, context, xattr)
cp -l file.txt link.txt             # Create hard link instead of copying
cp -s file.txt symlink.txt          # Create symbolic link instead of copying
```

### mv — Move or rename files

```bash
mv old.txt new.txt                  # Rename a file
mv file.txt /dest/dir/              # Move file to another directory
mv -i file.txt /dest/               # Interactive: prompt before overwriting
mv -n file.txt /dest/               # No-clobber: never overwrite
mv -v *.log /var/log/archive/       # Move with verbose output
mv -t /dest/ file1 file2 file3      # Move multiple files to target directory
```

### rm — Remove files and directories

```bash
rm file.txt                         # Remove a file
rm -r directory/                    # Remove directory recursively
rm -rf directory/                   # Force remove without prompting
rm -i *.tmp                         # Interactive: confirm each deletion
rm -v file.txt                      # Verbose: show what's being removed
rm -- -weirdname                    # Remove file starting with a dash
```

### find — Search for files in directory hierarchy

```bash
find / -name "*.conf"                           # Find by name (case-sensitive)
find / -iname "*.conf"                          # Find by name (case-insensitive)
find . -type f -name "*.go"                     # Find only regular files
find . -type d -name "test"                     # Find only directories
find . -type l                                  # Find only symlinks
find . -size +100M                              # Files larger than 100MB
find . -size -1k                                # Files smaller than 1KB
find . -mtime -7                                # Modified in last 7 days
find . -mmin -30                                # Modified in last 30 minutes
find . -newer reference.txt                     # Files newer than reference.txt
find . -user root                               # Files owned by root
find . -perm 644                                # Files with exact permissions 644
find . -perm -u+x                               # Files with user-execute permission
find . -empty                                   # Find empty files and directories
find . -name "*.log" -delete                    # Find and delete
find . -name "*.go" -exec grep -l "fmt" {} \;   # Find and execute command
find . -name "*.go" -exec grep -l "fmt" {} +    # Same but more efficient (batched)
find . -name "*.o" -o -name "*.tmp"             # OR condition
find . -maxdepth 2 -name "*.md"                 # Limit search depth
find . -not -path "./vendor/*" -name "*.go"     # Exclude paths
find . -type f -printf '%s %p\n' | sort -rn | head -10   # Top 10 largest files
```

### locate — Find files by name (using pre-built database)

```bash
locate myfile.conf              # Fast search using mlocate database
locate -i MyFile                # Case-insensitive search
locate -c "*.go"                # Count matching entries
locate -r '\.go$'               # Use regex
sudo updatedb                   # Rebuild the locate database
```

### stat — Display file status

```bash
stat file.txt                   # Full file metadata
stat -c '%a %U %G %s %n' *     # Custom format: permissions owner group size name
stat -c '%Y' file.txt           # Modification time as epoch
stat -f /                       # Filesystem status
stat -L symlink                 # Follow symlinks
```

### file — Determine file type

```bash
file binary                     # Identify file type (ELF, ASCII, etc.)
file -i document.pdf            # Show MIME type
file -s /dev/sda                # Examine special files (block devices)
file -z archive.gz              # Look inside compressed files
```

### ln — Create links

```bash
ln target.txt hardlink.txt      # Create a hard link (same inode)
ln -s /path/to/target symlink   # Create a symbolic link
ln -sf /new/target symlink      # Force-replace existing symlink
ln -sr target relative_link     # Create relative symlink
```

### chmod — Change file permissions

```bash
chmod 755 script.sh             # rwxr-xr-x
chmod 644 config.txt            # rw-r--r--
chmod u+x script.sh             # Add execute for user
chmod go-w file.txt             # Remove write for group and others
chmod a+r file.txt              # Add read for all
chmod -R 755 directory/         # Recursive permission change
chmod --reference=ref file.txt  # Copy permissions from another file
chmod 4755 program              # Set SUID bit
chmod 2755 directory            # Set SGID bit
chmod 1777 /tmp                 # Set sticky bit
```

### chown — Change file ownership

```bash
chown user:group file.txt       # Change owner and group
chown user file.txt             # Change owner only
chown :group file.txt           # Change group only
chown -R user:group dir/        # Recursive ownership change
chown --reference=ref file.txt  # Copy ownership from another file
```

### umask — Set default file permissions

```bash
umask                           # Display current umask
umask 022                       # New files: 644, new dirs: 755
umask 077                       # New files: 600, new dirs: 700
umask -S                        # Display in symbolic form
```

---

## 2. Text Processing

### cat — Concatenate and display files

```bash
cat file.txt                    # Display file contents
cat -n file.txt                 # Number all lines
cat -b file.txt                 # Number non-blank lines
cat -s file.txt                 # Squeeze repeated blank lines
cat -A file.txt                 # Show non-printing chars (tabs as ^I, line endings as $)
cat file1.txt file2.txt > merged.txt   # Concatenate files
cat << 'EOF' > script.sh        # Here document
#!/bin/bash
echo "Hello"
EOF
```

### grep — Search text using patterns

```bash
grep "pattern" file.txt                 # Search for pattern
grep -i "pattern" file.txt              # Case-insensitive
grep -r "pattern" /path/                # Recursive search
grep -rn "TODO" src/                    # Recursive with line numbers
grep -v "pattern" file.txt              # Invert match (exclude lines)
grep -c "pattern" file.txt              # Count matching lines
grep -l "pattern" *.go                  # List files containing pattern
grep -L "pattern" *.go                  # List files NOT containing pattern
grep -w "word" file.txt                 # Whole word match
grep -A 3 "error" log.txt              # 3 lines after match
grep -B 3 "error" log.txt              # 3 lines before match
grep -C 3 "error" log.txt              # 3 lines before and after
grep -E "regex|pattern" file.txt        # Extended regex (egrep)
grep -P '\d{3}-\d{4}' file.txt         # Perl-compatible regex
grep -o "pattern" file.txt              # Print only the matched part
grep --include="*.go" -r "func main" .  # Restrict to certain file types
grep --exclude-dir=vendor -r "TODO" .   # Exclude directories
```

### sed — Stream editor

```bash
sed 's/old/new/' file.txt               # Replace first occurrence per line
sed 's/old/new/g' file.txt              # Replace all occurrences
sed -i 's/old/new/g' file.txt           # In-place edit
sed -i.bak 's/old/new/g' file.txt      # In-place with backup
sed -n '10,20p' file.txt               # Print lines 10-20
sed '5d' file.txt                       # Delete line 5
sed '/pattern/d' file.txt              # Delete lines matching pattern
sed '/^$/d' file.txt                   # Delete empty lines
sed 's/^/PREFIX: /' file.txt           # Add prefix to each line
sed '1i\Header Line' file.txt          # Insert line before line 1
sed '$a\Footer Line' file.txt          # Append line after last line
sed -n '/START/,/END/p' file.txt       # Print lines between patterns
sed 's/\(.*\)/\U\1/' file.txt          # Convert to uppercase (GNU sed)
sed -e 's/a/b/' -e 's/c/d/' file.txt   # Multiple operations
```

### awk — Pattern scanning and processing

```bash
awk '{print $1}' file.txt                          # Print first column
awk '{print $NF}' file.txt                         # Print last column
awk -F: '{print $1, $3}' /etc/passwd               # Custom field separator
awk 'NR==5' file.txt                               # Print line 5
awk 'NR>=10 && NR<=20' file.txt                    # Print lines 10-20
awk '/pattern/' file.txt                           # Print lines matching pattern
awk '{sum += $1} END {print sum}' data.txt         # Sum a column
awk '{sum += $1} END {print sum/NR}' data.txt      # Average a column
awk 'length > 80' file.txt                         # Lines longer than 80 chars
awk '!seen[$0]++' file.txt                         # Remove duplicate lines (preserves order)
awk '{print NR": "$0}' file.txt                    # Number lines
awk -v OFS='\t' '{print $1, $3}' file.txt          # Set output field separator
awk 'BEGIN{ORS=","} {print $1}' file.txt           # Join values with comma
awk '{a[$1]+=$2} END{for(k in a) print k,a[k]}' f # Group and sum
```

### sort — Sort lines

```bash
sort file.txt                   # Alphabetical sort
sort -n file.txt                # Numeric sort
sort -r file.txt                # Reverse sort
sort -k2 file.txt               # Sort by second field
sort -k2,2n file.txt            # Numeric sort by second field only
sort -t: -k3 -n /etc/passwd     # Sort /etc/passwd by UID
sort -u file.txt                # Sort and remove duplicates
sort -h file.txt                # Human-readable numeric sort (1K, 2M, 3G)
sort -R file.txt                # Random shuffle
sort --stable -k1,1 file.txt    # Stable sort (preserve original order of equal elements)
```

### uniq — Report or omit repeated lines

```bash
uniq file.txt                   # Remove adjacent duplicates (sort first!)
uniq -c file.txt                # Count occurrences
uniq -d file.txt                # Show only duplicates
uniq -u file.txt                # Show only unique lines
sort file.txt | uniq -c | sort -rn   # Frequency count (most common first)
```

### wc — Word, line, character count

```bash
wc file.txt                     # Lines, words, bytes
wc -l file.txt                  # Count lines only
wc -w file.txt                  # Count words only
wc -c file.txt                  # Count bytes only
wc -m file.txt                  # Count characters
find . -name "*.go" | wc -l     # Count Go source files
```

### head / tail — View beginning or end of files

```bash
head file.txt                   # First 10 lines
head -n 20 file.txt             # First 20 lines
head -c 100 file.txt            # First 100 bytes
tail file.txt                   # Last 10 lines
tail -n 20 file.txt             # Last 20 lines
tail -f /var/log/syslog         # Follow file (live updates)
tail -F /var/log/syslog         # Follow with retry (handles log rotation)
tail -n +5 file.txt             # Everything from line 5 onward
```

### cut — Remove sections from lines

```bash
cut -d: -f1 /etc/passwd         # Extract first field (colon-delimited)
cut -d, -f1,3 data.csv          # Fields 1 and 3 from CSV
cut -c1-10 file.txt             # First 10 characters of each line
cut -d' ' -f2- file.txt         # Everything from field 2 onward
```

### tr — Translate or delete characters

```bash
tr 'a-z' 'A-Z' < file.txt      # Convert to uppercase
tr -d '\r' < dos.txt > unix.txt # Remove carriage returns (DOS → Unix)
tr -s ' ' < file.txt            # Squeeze repeated spaces
tr ':' '\n' <<< "$PATH"         # Show PATH entries one per line
tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c 32   # Random password
```

### tee — Read stdin and write to both stdout and files

```bash
command | tee output.log                  # Display and save output
command | tee -a output.log               # Append instead of overwrite
command 2>&1 | tee output.log             # Capture stderr too
command | tee file1.txt file2.txt         # Write to multiple files
```

### xargs — Build and execute commands from stdin

```bash
find . -name "*.tmp" | xargs rm                     # Delete found files
find . -name "*.tmp" -print0 | xargs -0 rm          # Handle filenames with spaces
echo "a b c" | xargs -n1                             # One argument per execution
cat urls.txt | xargs -I{} curl -O {}                 # Substitute placeholder
find . -name "*.go" | xargs -P4 -I{} go vet {}      # Parallel execution (4 procs)
seq 1 100 | xargs -n10                               # Group 10 per line
```

---

## 3. Process Management

### ps — Report process status

```bash
ps aux                          # All processes, BSD-style
ps -ef                          # All processes, POSIX-style
ps -eo pid,ppid,cmd,%mem,%cpu   # Custom columns
ps -eo pid,ppid,cmd --sort=-%mem | head -20   # Top memory consumers
ps -p 1234                      # Specific PID
ps -u username                  # Processes by user
ps --forest                     # Process tree (ASCII art)
ps -eo pid,tid,class,rtprio,ni,pri,psr,pcpu,stat,comm   # Scheduling info
ps -T -p <pid>                  # Show threads of a process
```

### top / htop — Dynamic process viewers

```bash
top                             # Interactive process viewer
top -b -n1                      # Batch mode, one iteration (for scripts)
top -p 1234                     # Monitor specific PID
top -u username                 # Filter by user
# Inside top: M=sort by memory, P=sort by CPU, k=kill, 1=per-CPU view, H=threads

htop                            # Enhanced interactive viewer
htop -p 1234,5678               # Monitor specific PIDs
htop -u username                # Filter by user
htop -t                         # Tree view
```

### kill / killall / pkill — Signal processes

```bash
kill 1234                       # Send SIGTERM (graceful shutdown)
kill -9 1234                    # Send SIGKILL (force kill)
kill -STOP 1234                 # Pause process
kill -CONT 1234                 # Resume paused process
kill -0 1234                    # Check if process exists (no signal sent)
kill -l                         # List all signals
pkill -f "pattern"              # Kill by command-line pattern
pkill -u username               # Kill all processes by user
pgrep -la "nginx"               # List PIDs and command lines matching pattern
```

### nice / renice — Manage process priority

```bash
nice -n 19 ./cpu_heavy_task     # Start with lowest priority
nice -n -20 ./critical_task     # Start with highest priority (needs root)
renice -n 10 -p 1234            # Change priority of running process
renice -n 5 -u username         # Change priority for all user's processes
```

### nohup / bg / fg / jobs — Job control

```bash
nohup ./long_task &             # Run immune to hangups, in background
jobs                            # List background jobs
fg %1                           # Bring job 1 to foreground
bg %1                           # Resume stopped job in background
# Ctrl+Z suspends the current foreground process
disown %1                       # Detach job from shell (survives logout)
```

### lsof — List open files

```bash
lsof -p 1234                    # Files opened by PID
lsof -i :8080                   # Processes using port 8080
lsof -i tcp                     # All TCP connections
lsof -u username                # Files opened by user
lsof +D /var/log/               # Files open in directory (non-recursive)
lsof -i @192.168.1.1            # Connections to specific host
lsof -c nginx                   # Files opened by command name
lsof /path/to/file              # Who has this file open?
```

### strace / ltrace — Trace system calls and library calls

```bash
strace ls                       # Trace system calls of a command
strace -p 1234                  # Attach to running process
strace -c ls                    # Summary of syscall counts and times
strace -e trace=open,read ls    # Trace only specific syscalls
strace -e trace=network ping -c1 google.com   # Network-related syscalls only
strace -f -e trace=clone,fork bash -c "ls | grep foo"  # Follow child processes
strace -T ls                    # Show time spent in each syscall
strace -t ls                    # Timestamp each syscall
strace -o output.txt ls         # Write trace to file

ltrace ls                       # Trace library calls
ltrace -c ls                    # Library call summary
ltrace -e malloc+free ls        # Trace specific library functions
```

---

## 4. System Information

### uname — System identification

```bash
uname -a                        # All system info
uname -r                        # Kernel release
uname -m                        # Machine architecture (x86_64, aarch64)
uname -n                        # Hostname
uname -s                        # Kernel name (Linux)
```

### hostname — Show or set system hostname

```bash
hostname                        # Display hostname
hostname -I                     # Display all IP addresses
hostname -f                     # Fully qualified domain name
hostnamectl                     # systemd hostname info and management
```

### uptime — System uptime and load

```bash
uptime                          # Uptime, users, load averages
uptime -p                       # Pretty format ("up 2 days, 3 hours")
uptime -s                       # System boot time
cat /proc/loadavg               # Load averages and process counts
```

### free — Memory usage

```bash
free                            # Memory in kilobytes
free -h                         # Human-readable
free -m                         # In megabytes
free -s 2                       # Refresh every 2 seconds
free -t                         # Show total row
```

### df / du — Disk usage

```bash
df -h                           # Filesystem disk space (human-readable)
df -i                           # Inode usage
df -T                           # Show filesystem type
df -h /home                     # Specific mount point

du -sh directory/               # Total size of directory
du -sh *                        # Size of each item in current dir
du -h --max-depth=1 /           # One-level depth summary
du -ah . | sort -rh | head -20  # Top 20 largest files/dirs
```

### lscpu / lsblk / lspci / dmidecode — Hardware info

```bash
lscpu                           # CPU architecture info
lscpu -e                        # Extended per-CPU info
lsblk                           # Block devices tree
lsblk -f                        # Show filesystems on devices
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
lspci                           # PCI devices
lspci -v                        # Verbose PCI info
lspci -k                        # Show kernel drivers in use
sudo dmidecode --type memory    # RAM module details
sudo dmidecode --type bios      # BIOS info
cat /proc/cpuinfo               # Detailed CPU info
cat /proc/meminfo               # Detailed memory info
```

---

## 5. Networking

### ip — Network configuration (iproute2)

```bash
ip addr show                    # Show all addresses
ip addr show eth0               # Show addresses for specific interface
ip addr add 192.168.1.10/24 dev eth0    # Add IP address
ip addr del 192.168.1.10/24 dev eth0    # Remove IP address
ip link show                    # Show network interfaces
ip link set eth0 up             # Bring interface up
ip link set eth0 down           # Bring interface down
ip route show                   # Show routing table
ip route add default via 192.168.1.1    # Add default gateway
ip route add 10.0.0.0/8 via 192.168.1.1 dev eth0   # Add static route
ip neigh show                   # Show ARP table (neighbors)
ip -s link show eth0            # Interface statistics
ip -j addr show | jq            # JSON output (for scripting)
```

### ss — Socket statistics (replacement for netstat)

```bash
ss -tlnp                        # TCP listening sockets with process info
ss -ulnp                        # UDP listening sockets
ss -s                           # Socket summary statistics
ss -t state established         # Only established TCP connections
ss -t dst 192.168.1.1           # Connections to specific destination
ss -t sport = :80               # Connections on source port 80
ss -tp                          # TCP connections with process names
ss -o state time-wait           # TIME_WAIT connections
```

### netstat — Network statistics (legacy but common)

```bash
netstat -tlnp                   # TCP listening ports with PID
netstat -an                     # All connections, numeric
netstat -s                      # Protocol statistics
netstat -r                      # Routing table
netstat -i                      # Interface statistics
```

### ping / traceroute — Network diagnostics

```bash
ping -c 4 google.com            # Send 4 ICMP echo requests
ping -i 0.2 google.com          # 200ms interval (needs root)
ping -s 1472 -M do google.com   # MTU path discovery
traceroute google.com           # Trace packet route (UDP)
traceroute -T google.com        # Use TCP SYN
traceroute -I google.com        # Use ICMP
mtr google.com                  # Combined ping + traceroute (live)
```

### dig / nslookup — DNS queries

```bash
dig google.com                  # Query A record
dig google.com MX               # Query MX records
dig google.com ANY              # Query all records
dig +short google.com           # Short answer only
dig +trace google.com           # Full delegation trace
dig @8.8.8.8 google.com        # Query specific DNS server
dig -x 8.8.8.8                 # Reverse DNS lookup
nslookup google.com             # Simple DNS lookup
```

### curl / wget — Transfer data from URLs

```bash
curl https://example.com                # GET request
curl -o file.zip https://url/file.zip   # Download to file
curl -O https://url/file.zip            # Download preserving remote name
curl -I https://example.com             # HEAD request (headers only)
curl -X POST -d '{"key":"val"}' -H 'Content-Type: application/json' https://api/endpoint
curl -s https://api/data | jq           # Silent mode + JSON processing
curl -u user:pass https://api/          # Basic auth
curl -k https://self-signed/            # Skip TLS verification
curl -L https://redirect.example.com    # Follow redirects
curl -w '%{http_code}\n' -s -o /dev/null https://example.com   # Just the status code
curl --connect-timeout 5 --max-time 10 https://slow.api/       # Timeouts
curl -x http://proxy:8080 https://example.com                  # Via proxy

wget https://example.com/file.tar.gz                           # Download file
wget -c https://example.com/large.iso                          # Resume download
wget -r -l2 https://example.com/docs/                          # Recursive download (depth 2)
wget --mirror https://example.com                              # Mirror entire site
```

### tcpdump — Packet capture

```bash
tcpdump -i eth0                         # Capture on interface
tcpdump -i any port 80                  # HTTP traffic on all interfaces
tcpdump -i eth0 host 192.168.1.1        # Traffic to/from specific host
tcpdump -i eth0 -w capture.pcap         # Write to pcap file
tcpdump -r capture.pcap                 # Read pcap file
tcpdump -i eth0 'tcp port 80 and host 10.0.0.1'  # Combined filter
tcpdump -i eth0 -n -c 100              # No DNS resolution, capture 100 packets
tcpdump -i eth0 -A port 80             # Print packet payload as ASCII
tcpdump -i eth0 -X port 443            # Hex + ASCII dump
```

### iptables / nftables — Firewall management

```bash
# iptables (legacy, still widely used)
iptables -L -n -v                       # List all rules with counters
iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # Allow inbound HTTP
iptables -A INPUT -s 10.0.0.0/8 -j DROP          # Block a subnet
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080  # Port redirect
iptables-save > rules.txt               # Export rules
iptables-restore < rules.txt            # Import rules

# nftables (modern replacement)
nft list ruleset                        # List all rules
nft add table inet filter               # Create table
nft add chain inet filter input '{ type filter hook input priority 0; policy accept; }'
nft add rule inet filter input tcp dport 80 accept
```

### nc (netcat) / socat — Network Swiss-army knives

```bash
nc -zv host 80                          # Port scan (check if port is open)
nc -l 8080                              # Listen on port 8080
echo "hello" | nc host 8080             # Send data to host:8080
nc -l 8080 > received_file              # Receive a file
nc host 8080 < send_file                # Send a file

socat TCP-LISTEN:8080,fork TCP:remote:80          # TCP proxy/forwarder
socat - TCP:host:80                               # Connect to host:80 interactively
socat TCP-LISTEN:8080,reuseaddr,fork EXEC:./handler.sh   # Simple server
```

---

## 6. Disk and Filesystem

### mount / umount — Filesystem mounting

```bash
mount                           # Show all mounted filesystems
mount /dev/sdb1 /mnt/usb        # Mount a device
mount -t ext4 /dev/sdb1 /mnt    # Mount specifying filesystem type
mount -o ro /dev/sdb1 /mnt      # Mount read-only
mount -o remount,rw /           # Remount root as read-write
mount --bind /src /dst          # Bind mount (directory alias)
umount /mnt/usb                 # Unmount
umount -l /mnt/busy             # Lazy unmount (detach now, cleanup when idle)
cat /etc/fstab                  # Persistent mount configuration
findmnt                         # Tree of mounted filesystems
```

### fdisk / mkfs / fsck — Disk partitioning and filesystem tools

```bash
sudo fdisk -l                   # List all disks and partitions
sudo fdisk /dev/sdb             # Interactive partitioning
sudo mkfs.ext4 /dev/sdb1        # Create ext4 filesystem
sudo mkfs.xfs /dev/sdb1         # Create XFS filesystem
sudo fsck /dev/sdb1             # Check and repair filesystem
sudo fsck -y /dev/sdb1          # Auto-repair (answer yes to all)
sudo e2fsck -f /dev/sdb1        # Force ext4 check
```

### blkid / tune2fs — Filesystem metadata

```bash
sudo blkid                      # Show block device attributes (UUID, TYPE)
sudo blkid /dev/sda1            # Specific device
sudo tune2fs -l /dev/sda1       # ext filesystem parameters
sudo tune2fs -c 30 /dev/sda1    # Set max mount count before check
sudo tune2fs -m 1 /dev/sda1     # Set reserved block percentage to 1%
```

---

## 7. Package Management

### apt — Debian/Ubuntu package management

```bash
sudo apt update                         # Refresh package index
sudo apt upgrade                        # Upgrade all packages
sudo apt install nginx                  # Install a package
sudo apt remove nginx                   # Remove package (keep config)
sudo apt purge nginx                    # Remove package and config
sudo apt autoremove                     # Remove unused dependencies
apt search keyword                      # Search for packages
apt show nginx                          # Show package details
apt list --installed                    # List installed packages
dpkg -l | grep nginx                    # Check if package is installed
dpkg -L nginx                           # List files installed by package
```

### yum / dnf — RHEL/CentOS/Fedora package management

```bash
sudo dnf check-update                   # Check for available updates
sudo dnf upgrade                        # Upgrade all packages
sudo dnf install httpd                   # Install a package
sudo dnf remove httpd                    # Remove a package
dnf search keyword                       # Search packages
dnf info httpd                           # Package info
dnf list installed                       # List installed packages
rpm -qa | grep httpd                     # Query installed RPMs
rpm -ql httpd                            # List files in RPM
```

### snap — Snap package management

```bash
snap list                                # List installed snaps
snap find keyword                        # Search for snaps
sudo snap install package                # Install a snap
sudo snap remove package                 # Remove a snap
sudo snap refresh                        # Update all snaps
snap info package                        # Snap details
```

---

## 8. User Management

```bash
sudo useradd -m -s /bin/bash username    # Create user with home dir and shell
sudo useradd -m -G sudo,docker user     # Create user with supplementary groups
sudo userdel -r username                 # Delete user and home directory
sudo passwd username                     # Set/change password
sudo usermod -aG docker username         # Add user to group
groups username                          # List user's groups
id username                              # UID, GID, groups
whoami                                   # Current username
who                                      # Logged-in users
w                                        # Logged-in users with activity
last                                     # Login history
su - username                            # Switch user (login shell)
sudo -i                                  # Root login shell
sudo -u www-data command                 # Run command as another user
visudo                                   # Safely edit sudoers file
```

---

## 9. Archiving and Compression

### tar — Tape archive

```bash
tar cf archive.tar files/               # Create archive (no compression)
tar czf archive.tar.gz files/           # Create with gzip compression
tar cjf archive.tar.bz2 files/         # Create with bzip2 compression
tar cJf archive.tar.xz files/          # Create with xz compression
tar xf archive.tar.gz                  # Extract (auto-detects compression)
tar xf archive.tar.gz -C /dest/        # Extract to specific directory
tar tf archive.tar.gz                  # List contents without extracting
tar czf archive.tar.gz --exclude='*.log' files/   # Exclude patterns
tar czf - files/ | ssh host 'tar xzf - -C /dest'  # Archive over SSH
```

### Individual compression tools

```bash
gzip file.txt                           # Compress (replaces original with file.txt.gz)
gzip -d file.txt.gz                     # Decompress
gzip -k file.txt                        # Keep original file
gzip -9 file.txt                        # Best compression

bzip2 file.txt                          # Better compression, slower
bzip2 -d file.txt.bz2                   # Decompress

xz file.txt                             # Best compression, slowest
xz -d file.txt.xz                       # Decompress
xz -9 -T0 file.txt                     # Best compression, all threads

zip -r archive.zip directory/           # Create zip
unzip archive.zip                       # Extract zip
unzip -l archive.zip                    # List contents
```

---

## 10. systemd

### systemctl — Service management

```bash
systemctl status nginx                  # Service status
systemctl start nginx                   # Start service
systemctl stop nginx                    # Stop service
systemctl restart nginx                 # Restart service
systemctl reload nginx                  # Reload configuration
systemctl enable nginx                  # Enable at boot
systemctl disable nginx                 # Disable at boot
systemctl is-active nginx               # Check if running
systemctl is-enabled nginx              # Check if enabled at boot
systemctl list-units --type=service     # List all services
systemctl list-units --state=failed     # List failed services
systemctl daemon-reload                 # Reload unit files after changes
systemctl mask nginx                    # Prevent service from being started
systemctl unmask nginx                  # Reverse mask
systemctl edit nginx --full             # Edit unit file
systemctl show nginx                    # Show all properties
systemctl list-dependencies nginx       # Show service dependencies
```

### journalctl — systemd journal (logs)

```bash
journalctl                              # All logs
journalctl -u nginx                     # Logs for specific service
journalctl -u nginx --since "1 hour ago"  # Recent logs
journalctl -u nginx -f                  # Follow (like tail -f)
journalctl -b                           # Logs since last boot
journalctl -b -1                        # Logs from previous boot
journalctl -p err                       # Only errors and above
journalctl -k                           # Kernel messages only
journalctl --disk-usage                 # Journal disk usage
journalctl --vacuum-size=500M           # Limit journal size
journalctl -o json-pretty               # JSON output
journalctl _PID=1234                    # Logs from specific PID
journalctl --since "2024-01-01" --until "2024-01-02"   # Date range
```

### Other systemd utilities

```bash
loginctl list-sessions                  # List user sessions
loginctl show-session <id>              # Session details
timedatectl                             # Time and timezone info
timedatectl set-timezone America/New_York  # Set timezone
timedatectl set-ntp true                # Enable NTP
hostnamectl                             # Hostname info
hostnamectl set-hostname myserver       # Set hostname
systemd-analyze                         # Boot time analysis
systemd-analyze blame                   # Per-service boot time
systemd-analyze critical-chain          # Critical path in boot
```

---

## 11. Performance Analysis

### vmstat — Virtual memory statistics

```bash
vmstat 1                        # Update every 1 second
vmstat 1 10                     # 10 samples, 1 second apart
vmstat -s                       # Memory statistics summary
vmstat -d                       # Disk statistics
# Key columns: r=running, b=blocked, si/so=swap in/out, us/sy/id=user/sys/idle CPU
```

### iostat — I/O statistics

```bash
iostat                          # CPU and disk I/O summary
iostat -x 1                     # Extended stats, every 1 second
iostat -d sda 1                 # Specific device
iostat -p sda                   # Partition stats
# Key columns: %util=device utilization, await=avg I/O wait (ms), r/s and w/s=IOPS
```

### mpstat — Per-CPU statistics

```bash
mpstat                          # Average CPU stats
mpstat -P ALL 1                 # Per-CPU stats every second
mpstat -P 0,1 1                 # Specific CPUs
```

### sar — System activity reporter

```bash
sar -u 1 5                      # CPU usage, 5 samples, 1 sec apart
sar -r 1 5                      # Memory usage
sar -b 1 5                      # I/O stats
sar -n DEV 1 5                  # Network interface stats
sar -q 1 5                      # Run queue and load average
sar -d 1 5                      # Block device activity
sar -f /var/log/sa/sa01          # Read historical data
```

### perf — Linux profiling tool

```bash
perf stat ./myprogram                   # Hardware counters for a command
perf record -g ./myprogram              # Record call-graph profile
perf report                             # Interactive report of recorded data
perf top                                # Live CPU profiling (like top for functions)
perf stat -e cache-misses ./prog        # Specific event
perf record -e cycles -g -p 1234        # Profile running process
perf list                               # List available events
```

### bpftrace — Dynamic tracing (eBPF)

```bash
bpftrace -e 'tracepoint:syscalls:sys_enter_open { printf("%s %s\n", comm, str(args->filename)); }'
bpftrace -e 'kprobe:vfs_read { @bytes = hist(arg2); }'     # Histogram of read sizes
bpftrace -e 'tracepoint:sched:sched_switch { @[comm] = count(); }'  # Context switch counts
bpftrace -l 'tracepoint:syscalls:*'                         # List available tracepoints
```

---

## 12. Container-Related

### Namespaces

```bash
lsns                                    # List all namespaces
lsns -t net                             # List network namespaces only
unshare --net --pid --fork --mount-proc bash   # Create new net+pid namespace
nsenter -t 1234 -n                      # Enter network namespace of PID
nsenter -t 1234 -m -u -i -n -p         # Enter all namespaces of PID
readlink /proc/self/ns/net              # Show current network namespace
ls -la /proc/1234/ns/                   # Namespaces of a process
```

### Network namespaces with ip

```bash
ip netns add myns                       # Create network namespace
ip netns list                           # List network namespaces
ip netns exec myns ip addr              # Run command in namespace
ip link add veth0 type veth peer name veth1   # Create veth pair
ip link set veth1 netns myns            # Move interface to namespace
ip netns exec myns ip addr add 10.0.0.1/24 dev veth1
ip netns exec myns ip link set veth1 up
ip netns del myns                       # Delete namespace
```

### Cgroups

```bash
# cgroups v2 (unified hierarchy)
cat /sys/fs/cgroup/cgroup.controllers           # Available controllers
ls /sys/fs/cgroup/                               # Cgroup tree root
cat /sys/fs/cgroup/user.slice/memory.current     # Memory usage of user slice
cat /sys/fs/cgroup/system.slice/cpu.stat         # CPU stats

# Create and configure a cgroup
mkdir /sys/fs/cgroup/mygroup
echo "100000 1000000" > /sys/fs/cgroup/mygroup/cpu.max    # 10% CPU limit
echo "256M" > /sys/fs/cgroup/mygroup/memory.max           # 256MB memory limit
echo $$ > /sys/fs/cgroup/mygroup/cgroup.procs             # Add current shell

# Inspect container cgroups
systemd-cgls                                     # Cgroup tree
systemd-cgtop                                    # Top-like view for cgroups
cat /proc/1234/cgroup                            # Cgroup membership of a process
```

### Useful /proc and /sys inspections

```bash
cat /proc/1234/status           # Process status (VmRSS, Threads, etc.)
cat /proc/1234/maps             # Memory mappings
cat /proc/1234/fd/              # Open file descriptors (ls -la)
cat /proc/1234/limits           # Resource limits
cat /proc/1234/oom_score        # OOM killer score
cat /proc/1234/net/tcp          # TCP connections (hex)
cat /proc/1234/mountinfo        # Mount information
cat /proc/1234/environ          # Environment variables (null-separated)
cat /proc/sys/vm/swappiness     # Swappiness setting
cat /proc/sys/net/ipv4/ip_forward  # IP forwarding status
sysctl -a                       # All kernel parameters
sysctl -w net.ipv4.ip_forward=1 # Set kernel parameter
```

---

## Quick Reference: Pipes & Redirections

```bash
cmd > file          # Redirect stdout to file (overwrite)
cmd >> file         # Redirect stdout to file (append)
cmd 2> file         # Redirect stderr to file
cmd 2>&1            # Redirect stderr to stdout
cmd &> file         # Redirect both stdout and stderr to file
cmd1 | cmd2         # Pipe stdout of cmd1 to stdin of cmd2
cmd1 |& cmd2        # Pipe both stdout and stderr
cmd < file          # Redirect file to stdin
cmd <<< "string"    # Here-string (feed string as stdin)
```

## Quick Reference: Bash Shortcuts & Tricks

```bash
!!                  # Repeat last command
!$                  # Last argument of previous command
!^                  # First argument of previous command
!*                  # All arguments of previous command
^old^new            # Replace 'old' with 'new' in last command
Ctrl+R              # Reverse search command history
Ctrl+A / Ctrl+E     # Move to start / end of line
Ctrl+U / Ctrl+K     # Cut from cursor to start / end of line
Ctrl+W              # Cut previous word
Ctrl+Y              # Paste cut text
Alt+.               # Insert last argument of previous command
```

---

*This cheatsheet covers the commands most relevant to systems programming and
infrastructure work. For full documentation, use `man <command>` or `<command> --help`.*
