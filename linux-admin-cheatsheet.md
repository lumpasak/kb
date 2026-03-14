# Linux / Cloudera Admin — Command Cheat Sheet

---

## 1. Finding Biggest Files & Directories

```bash
# Top 20 biggest files on the entire system
find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Top 20 biggest files under a specific path
find /var/log -type f -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Top 20 biggest directories (1 level deep)
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -20

# Recursive biggest directories under /var
du -ah /var 2>/dev/null | sort -rh | head -30

# Files bigger than 1 GB
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null

# Files bigger than 500 MB, show path + size
find / -type f -size +500M -printf '%s %p\n' 2>/dev/null | sort -rn | head -20

# Quick summary of disk usage per top-level dir
du -sh /* 2>/dev/null | sort -rh
```

---

## 2. Finding & Analyzing Logs

```bash
# Biggest log files
find /var/log -type f -name "*.log*" -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Cloudera logs specifically
find /var/log/hadoop* /var/log/hive* /var/log/hbase* /var/log/impala* \
     /var/log/spark* /var/log/ranger* /var/log/solr* /var/log/kafka* \
     /var/log/cloudera* -type f -exec du -h {} + 2>/dev/null | sort -rh | head -30

# CM agent/server logs
du -sh /var/log/cloudera-scm-agent/*
du -sh /var/log/cloudera-scm-server/*

# Files modified in the last 24h (active logs)
find /var/log -type f -mtime -1 -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Files NOT modified in 30+ days (stale logs, cleanup candidates)
find /var/log -type f -mtime +30 -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Watch a log in real-time
tail -f /var/log/hadoop-hdfs/hadoop-cmf-hdfs-NAMENODE-*.log.out

# Search for errors in logs
grep -rli "ERROR\|FATAL\|OutOfMemory\|Exception" /var/log/hadoop* 2>/dev/null

# Count errors per log file
for f in /var/log/hadoop-hdfs/*.log; do echo "$(grep -c ERROR "$f") $f"; done | sort -rn

# Journal logs for a service
journalctl -u cloudera-scm-agent --since "1 hour ago" --no-pager
journalctl -u cloudera-scm-server -f   # follow live
```

---

## 3. Disk Space & Filesystem

```bash
# Disk usage overview
df -hT                           # all filesystems with type
df -hT /                         # root partition only
df -i                            # inode usage (can run out before space!)

# Which partition is full?
df -h | awk '$5+0 > 80 {print}'  # partitions over 80%

# Quickly find what's eating space
ncdu /var/log                    # interactive disk browser (install: yum install ncdu)

# Check for deleted but open files (space not freed)
lsof +L1 2>/dev/null | grep deleted

# LVM / block device info
lsblk -f
pvs; vgs; lvs

# Extend LV if needed
lvextend -L +10G /dev/mapper/vg-lv_var
xfs_growfs /var                  # XFS
resize2fs /dev/mapper/vg-lv_var  # ext4
```

---

## 4. Network — Ports, Connections, Listeners

```bash
# All listening ports with process names
ss -tlnp                         # TCP listeners
ss -ulnp                         # UDP listeners
ss -tlnp | grep -E ':(7180|7182|8088|8888|10000|10002|21050|25010|9083|8020|9870)'

# Netstat alternative (if available)
netstat -tlnp

# Check if a specific port is open/listening
ss -tlnp | grep :7180            # CM server
ss -tlnp | grep :10000           # HiveServer2
ss -tlnp | grep :21050           # Impala daemon
ss -tlnp | grep :8088            # YARN ResourceManager
ss -tlnp | grep :9870            # HDFS NameNode WebUI
ss -tlnp | grep :8020            # HDFS NameNode RPC
ss -tlnp | grep :9083            # Hive Metastore

# All established connections
ss -tnp                          # all TCP connections
ss -tnp | grep ESTAB | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn  # top IPs

# Test connectivity to a remote port
nc -zv hostname 7180             # netcat
timeout 3 bash -c 'echo > /dev/tcp/hostname/7180' && echo OPEN || echo CLOSED

# Trace route / DNS
traceroute hostname
dig hostname
nslookup hostname
host hostname

# Firewall rules
iptables -L -n --line-numbers
firewall-cmd --list-all          # firewalld
```

### Key Cloudera Ports Reference

| Port  | Service                        |
|-------|--------------------------------|
| 7180  | CM Server (HTTP)               |
| 7183  | CM Server (HTTPS)              |
| 7182  | CM Agent → Server              |
| 8020  | HDFS NameNode RPC              |
| 9870  | HDFS NameNode Web UI           |
| 8088  | YARN ResourceManager           |
| 8042  | YARN NodeManager               |
| 10000 | HiveServer2 (Thrift)           |
| 10002 | HiveServer2 (HTTP/WebUI)       |
| 9083  | Hive Metastore                 |
| 21050 | Impala Daemon (Beeswax)        |
| 21000 | Impala Daemon (HiveServer2)    |
| 25010 | Impala Daemon WebUI            |
| 25020 | Impala StateStore WebUI        |
| 8889  | Hue                            |
| 8088  | Ranger Admin                   |
| 6188  | Ranger Admin (Alt)             |
| 9092  | Kafka Broker                   |
| 2181  | ZooKeeper                      |
| 8983  | Solr                           |
| 88    | Kerberos KDC                   |

---

## 5. Process Management

```bash
# Top processes by CPU
ps aux --sort=-%cpu | head -20

# Top processes by memory
ps aux --sort=-%mem | head -20

# Find process by name
ps aux | grep -i impala
pgrep -af "HiveServer2"

# Process tree
pstree -p <PID>

# What is process doing? (open files, connections)
lsof -p <PID>
ls -la /proc/<PID>/fd | wc -l   # file descriptor count

# Kill gracefully, then force
kill <PID>
kill -9 <PID>

# Interactive process monitor
top -c                           # press 'M' for memory sort, 'P' for CPU
htop                             # better interactive (install: yum install htop)

# Java processes (Cloudera services are Java)
jps -lm                          # list all JVMs
jstack <PID> > /tmp/thread_dump.txt     # thread dump
jmap -heap <PID>                        # heap summary
jstat -gcutil <PID> 1000 10             # GC stats every 1s, 10 times
```

---

## 6. Memory & Swap

```bash
# Memory overview
free -h
cat /proc/meminfo

# What's using the most memory?
ps aux --sort=-%mem | head -15

# Clear page cache (careful in production)
sync; echo 3 > /proc/sys/vm/drop_caches

# Swap usage per process
for f in /proc/*/status; do
  awk '/VmSwap|Name/{printf $2 " " $3}' "$f" 2>/dev/null; echo
done | sort -k2 -rn | head -20

# Check OOM killer logs
dmesg | grep -i "oom\|killed process"
journalctl -k | grep -i oom
```

---

## 7. User & Permission Management

```bash
# Current user info
id
whoami
groups

# List all users / groups
cat /etc/passwd
cat /etc/group
getent passwd                    # includes LDAP/AD users
getent group

# File permissions
ls -la /path
stat /path/to/file

# Change ownership & permissions
chown -R hdfs:hadoop /data
chmod -R 750 /data

# ACLs (important for Cloudera data dirs)
getfacl /path/to/dir
setfacl -m u:hive:rwx /path/to/dir
setfacl -R -m g:hadoop:rx /data

# Who is logged in
w
last -20                         # last 20 logins
lastlog                          # last login per user

# Sudo check
sudo -l                          # what can I sudo?
```

---

## 8. Kerberos (Cloudera)

```bash
# Initialize ticket
kinit admin@REALM.COM
kinit -kt /path/to/keytab principal@REALM

# Check current ticket
klist
klist -e                         # show encryption types

# Inspect keytab contents
klist -kt /path/to/keytab
klist -kte /path/to/keytab       # with encryption types

# Destroy ticket
kdestroy

# Test Kerberos auth
kinit -kt /etc/security/keytabs/hdfs.keytab hdfs/hostname@REALM
hdfs dfs -ls /

# Kadmin (admin operations)
kadmin -q "listprincs"
kadmin -q "getprinc hdfs/hostname@REALM"
```

---

## 9. HDFS Commands

```bash
# Disk usage in HDFS
hdfs dfs -du -h /                           # top level
hdfs dfs -du -h -s /user/*                  # per user
hdfs dfs -du -h /tmp | sort -rh | head -20  # biggest in /tmp

# HDFS report (block, capacity info)
hdfs dfsadmin -report
hdfs dfsadmin -report | head -50

# Safe mode
hdfs dfsadmin -safemode get
hdfs dfsadmin -safemode leave

# FSCK
hdfs fsck / -files -blocks -locations       # full check
hdfs fsck /path -list-corruptfileblocks

# Trash
hdfs dfs -expunge                           # empty trash

# Snapshot management
hdfs dfsadmin -allowSnapshot /path
hdfs dfs -createSnapshot /path snap_name
hdfs dfs -ls /path/.snapshot/

# Balance
hdfs balancer -threshold 5
```

---

## 10. Service Management (systemd)

```bash
# Cloudera services
systemctl status cloudera-scm-agent
systemctl status cloudera-scm-server
systemctl restart cloudera-scm-agent
systemctl restart cloudera-scm-server

# Generic service ops
systemctl list-units --type=service --state=running
systemctl list-units --type=service --state=failed
systemctl enable <service>
systemctl disable <service>

# Check boot targets
systemctl get-default
systemctl list-dependencies multi-user.target
```

---

## 11. SSL/TLS & Certificates

```bash
# Check certificate expiration
openssl x509 -in /path/cert.pem -noout -enddate -subject

# Check all certs in a directory
for f in /path/*.pem; do echo "=== $f ==="; openssl x509 -in "$f" -noout -enddate -subject; done

# Check remote server certificate
openssl s_client -connect hostname:7183 -showcerts </dev/null 2>/dev/null | openssl x509 -noout -enddate -subject

# Inspect JKS keystore (Cloudera uses these)
keytool -list -v -keystore /path/keystore.jks -storepass <password>
keytool -list -v -keystore /path/truststore.jks -storepass <password>

# Check cert chain
openssl s_client -connect hostname:port -CAfile /path/ca.pem

# Convert PEM to JKS
keytool -importcert -file cert.pem -keystore truststore.jks -alias myalias -storepass changeit
```

---

## 12. Cron & Scheduling

```bash
# List cron jobs
crontab -l                       # current user
crontab -l -u hdfs               # specific user
ls -la /etc/cron.d/              # system cron files
cat /etc/crontab

# Edit crontab
crontab -e

# Check if cron ran
grep CRON /var/log/cron
journalctl -u crond --since "1 hour ago"
```

---

## 13. System Info & Hardware

```bash
# OS info
cat /etc/os-release
uname -a
hostnamectl

# CPU
lscpu
nproc

# Memory
free -h
dmidecode -t memory 2>/dev/null | grep -i size

# Disk hardware
lsblk
fdisk -l
smartctl -a /dev/sda             # SMART health

# Network interfaces
ip a
ip r                             # routing table

# Uptime & load
uptime
cat /proc/loadavg

# System logs
dmesg | tail -50
journalctl -p err --since "24 hours ago"
```

---

## 14. Compression & Archives

```bash
# Tar
tar czf archive.tar.gz /path/to/dir
tar xzf archive.tar.gz
tar tzf archive.tar.gz           # list contents

# Find and compress old logs
find /var/log -name "*.log" -mtime +7 -exec gzip {} \;

# Check compressed file size
ls -lhS /var/log/**/*.gz | head -20
```

---

## 15. Handy One-Liners

```bash
# Which process is using a file?
fuser -v /path/to/file
lsof /path/to/file

# Which process is using a port?
fuser -v 7180/tcp
lsof -i :7180

# Find recently modified files (last 10 min)
find /etc -mmin -10 -type f

# Find files owned by a user
find / -user hdfs -type f 2>/dev/null | head -20

# Disk I/O monitoring
iostat -xz 1 5                   # 5 samples, 1s interval
iotop                            # interactive (yum install iotop)

# Network throughput
iftop                            # interactive (yum install iftop)
nload                            # per-interface bandwidth

# Diff two configs
diff /etc/hadoop/conf/core-site.xml /etc/hadoop/conf.backup/core-site.xml

# Watch a command repeatedly
watch -n 5 'df -h | grep -E "/$|/var"'
watch -n 2 'ss -tnp | grep :7180 | wc -l'

# Parallel SSH to multiple nodes
for h in node1 node2 node3; do echo "=== $h ==="; ssh $h 'df -h / && free -h'; done
```

---

## 16. Ranger / Solr (Cloudera-specific cleanup)

```bash
# Ranger audit spool (common disk killer)
du -sh /var/lib/ranger/*/audit_spool/
find /var/lib/ranger/*/audit_spool/ -type f -mtime +7 -delete

# Solr data size
du -sh /var/solr/data/
du -sh /var/lib/solr/

# Clean old Solr audit logs (via Solr API or CM config)
# Set ranger.audit.solr.config.ttl in CM → Ranger → Configs
```
