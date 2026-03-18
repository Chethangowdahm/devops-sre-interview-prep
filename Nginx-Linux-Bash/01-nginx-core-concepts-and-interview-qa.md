# Nginx, Linux & Bash - Interview Q&A
> Senior DevOps/SRE Level

---

# PART 1: NGINX

### What is Nginx?
High-performance web server, reverse proxy, load balancer. Event-driven, asynchronous, handles 10k+ concurrent connections with low memory.

### Production Nginx Config
```nginx
worker_processes auto;

http {
    log_format main '$remote_addr - [$time_local] "$request" $status $body_bytes_sent';
    access_log /var/log/nginx/access.log main;

    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    upstream api_backend {
        least_conn;
        server app1:8080;
        server app2:8080;
        keepalive 32;
    }

    server {
        listen 80;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name example.com;

        ssl_certificate     /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols       TLSv1.2 TLSv1.3;

        add_header Strict-Transport-Security 'max-age=31536000' always;

        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://api_backend;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_connect_timeout 10s;
            proxy_read_timeout 30s;
        }
    }
}
```

### Load Balancing Algorithms
- **round_robin** (default): Distribute evenly
- **least_conn**: Send to server with fewest connections
- **ip_hash**: Same client to same server (session affinity)
- **weight**: Manual weighting (weight=3)

### Interview Q: Nginx vs Apache?
Nginx: Event-driven, handles 10k+ connections with low memory. Better for static files, high concurrency.
Apache: Thread-per-connection, better ecosystem for dynamic content via modules.
Production: Use Nginx for most modern workloads.

### Common Nginx Issues
```bash
# Test config before reload
nginx -t

# Graceful reload (zero downtime)
nginx -s reload

# Check error logs
tail -f /var/log/nginx/error.log

# Check access patterns
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
# Shows HTTP status code distribution

# Top IPs
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

---

# PART 2: LINUX TROUBLESHOOTING

## Essential Commands

```bash
# CPU - which process is using it?
top                              # Press P for CPU sort
ps aux --sort=-%cpu | head -10
htop                             # Better top

# Memory
free -h
vmstat 1 5                       # 5 snapshots every 1 second
cat /proc/meminfo

# Disk
df -h                            # Disk usage
du -sh /* 2>/dev/null | sort -rh | head -20  # Largest dirs
iostat -x 1                      # Disk I/O
lsof | grep deleted              # Deleted files still open (inode leak)

# Network
ss -tulpn                        # Listening ports (faster than netstat)
tcpdump -i eth0 port 80 -w capture.pcap

# Process
strace -p <pid>                  # System calls
lsof -p <pid>                    # Open files

# Logs
journalctl -u myservice -n 100 --since '1h ago'
tail -f /var/log/syslog
grep -rn 'ERROR' /var/log/myapp/ --include='*.log'
```

## Troubleshooting Scenarios

### High CPU
```bash
# 1. Find process
ps aux --sort=-%cpu | head -5
# 2. What is it doing?
strace -p <pid>
# 3. Is it a loop? Check code or kill
kill -SIGUSR1 <pid>  # Graceful signal for stack dump
```

### Disk Space Full
```bash
# 1. Where is space used?
du -sh /* | sort -rh | head -10
# 2. Large files
find / -type f -size +1G 2>/dev/null
# 3. Old logs
du -sh /var/log/*
# 4. Deleted but open files
lsof | grep '(deleted)' | awk '{print $7}' | sort -rn | head
# 5. Inodes full?
df -i
```

### Port Not Accessible
```bash
# Is service listening?
ss -tulpn | grep :8080
# Is firewall blocking?
iptables -L -n | grep 8080
# Is it reachable?
curl -v http://localhost:8080/health
telnet localhost 8080
```

---

# PART 3: BASH SCRIPTING

```bash
#!/bin/bash
set -euo pipefail  # Exit on error, treat unset vars as error, catch pipe failures

# Logging function
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
error_exit() { echo "ERROR: $*" >&2; exit 1; }

# Check dependencies
command -v kubectl >/dev/null 2>&1 || error_exit 'kubectl not found'

# Variables
NAMESPACE=${1:-default}  # First arg or default
TODAY=$(date +%Y%m%d)

# Loops
for pod in $(kubectl get pods -n "$NAMESPACE" -o name); do
    log "Checking $pod"
    kubectl logs "$pod" -n "$NAMESPACE" --tail=10
done

# Conditionals
if kubectl get deployment myapp -n "$NAMESPACE" &>/dev/null; then
    log 'Deployment exists'
else
    error_exit 'Deployment not found'
fi

# Trap for cleanup
cleanup() { log 'Cleaning up'; rm -f /tmp/tmpfile; }
trap cleanup EXIT

# Array
SERVICES=('api' 'worker' 'scheduler')
for svc in "${SERVICES[@]}"; do
    log "Processing $svc"
done

# String manipulation
FILE='/var/log/app.log'
BASE=$(basename "$FILE" .log)    # app
DIR=$(dirname "$FILE")           # /var/log

# Capture command output
PODS=$(kubectl get pods -n production -o jsonpath='{.items[*].metadata.name}')

log 'Script completed successfully'
```

## Common Bash One-liners
```bash
# Count HTTP 5xx errors in nginx log
awk '$9 ~ /^5/' /var/log/nginx/access.log | wc -l

# Top 10 slowest requests
awk '{print $NF, $0}' /var/log/nginx/access.log | sort -rn | head -10

# Find all processes using >10% CPU
ps aux | awk '$3 > 10 {print $0}'

# Check all services status
for s in nginx mysql redis; do systemctl status $s | grep -E 'active|inactive'; done

# Kill all processes by name
pkill -f 'python my_script.py'

# Watch log in real-time and grep
tail -f /var/log/app.log | grep --line-buffered 'ERROR'
```
