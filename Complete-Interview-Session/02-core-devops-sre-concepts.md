# Complete Interview Session — File 2: Core DevOps & SRE Concepts Round

> **Round Type:** Foundational technical screening — tests depth of understanding, not just tool knowledge.
> **Duration:** Usually 30–45 minutes
> **Tone:** Be confident. Use real examples. Show you think in systems, not just commands.

---

## SECTION 1: LINUX & OPERATING SYSTEM

---

### Q1. "A production server is running slow. Walk me through how you diagnose it."

**[WHY THEY ASK]:** Tests systematic thinking and Linux proficiency under pressure.

**[YOUR ANSWER]:**
"I follow a structured top-down approach:

**Step 1 — CPU:**
```bash
top -c          # See overall CPU usage and top processes
mpstat -P ALL 1 # Per-core CPU utilization
ps aux --sort=-%cpu | head -10  # Top CPU consumers
```

**Step 2 — Memory:**
```bash
free -h         # RAM and swap usage
vmstat 1 5      # Memory, swap, I/O, CPU summary
cat /proc/meminfo | grep -i available
```

**Step 3 — Disk I/O:**
```bash
iostat -xz 1    # Disk I/O with utilization
iotop           # Per-process disk I/O
df -h           # Disk space
lsof | wc -l    # Open file descriptors
```

**Step 4 — Network:**
```bash
ss -tulnp       # Open ports and connections
netstat -s      # Network statistics
sar -n DEV 1 5  # Network throughput per interface
```

**Step 5 — Application logs:**
```bash
journalctl -u myapp --since '10 minutes ago'
tail -f /var/log/app/error.log
dmesg | tail -20  # Kernel messages — OOM killer, hardware errors
```

At Vendasta, we had a GCP VM running slow. `iostat` showed 100% iowait on the disk. The culprit was a runaway logging process filling disk with debug logs. We fixed it by rotating logs, adjusting log levels, and setting up a disk utilization alert in Datadog."

---

### Q2. "Explain the Linux boot process."

**[YOUR ANSWER]:**
"The Linux boot process has 6 stages:

1. **BIOS/UEFI** — firmware initializes hardware, performs POST, finds bootable device
2. **Bootloader (GRUB2)** — loads the kernel image into memory
3. **Kernel initialization** — kernel decompresses itself, initializes memory, detects hardware
4. **initrd/initramfs** — temporary root filesystem, loads necessary drivers
5. **systemd (init process PID 1)** — starts all system services according to targets
6. **User space** — login prompt or graphical interface

In production, this matters when a server fails to boot after a kernel upgrade — I've had to use rescue mode and `grub2-mkconfig` to fix broken bootloaders."

---

### Q3. "What is the difference between a process and a thread?"

**[YOUR ANSWER]:**
"A **process** is an independent program instance with its own memory space, file descriptors, and PID. Processes are isolated from each other — one crashing doesn't affect others.

A **thread** is a lightweight unit of execution within a process. Threads share the same memory space and file descriptors of their parent process. They are faster to create and communicate, but a bug in one thread can crash the entire process.

In the context of SRE, this matters for capacity planning — a multi-threaded application uses fewer system resources than the equivalent multi-process approach, but requires careful concurrency handling to avoid race conditions."

---

### Q4. "Explain file permissions in Linux. What does 755 mean?"

**[YOUR ANSWER]:**
"Linux permissions have three groups: Owner, Group, Others. Each group has three bits: Read (4), Write (2), Execute (1).

755 means:
- Owner: 7 = 4+2+1 = Read + Write + Execute
- Group: 5 = 4+0+1 = Read + Execute
- Others: 5 = 4+0+1 = Read + Execute

This is typical for executable scripts or web directories. In production, I've seen security incidents caused by world-writable files (777). I always enforce least-privilege — SSH keys are 600, directories 700 or 750 at most."

---

## SECTION 2: NETWORKING

---

### Q5. "What happens when you type 'google.com' in a browser?"

**[WHY THEY ASK]:** Tests full-stack networking knowledge — DNS, TCP, HTTP.

**[YOUR ANSWER]:**
"This covers the full networking stack:

1. **DNS Resolution:** Browser checks its cache → OS cache → /etc/hosts → configured DNS server. The DNS server resolves google.com to an IP via recursive lookup (root → TLD → authoritative nameserver).
2. **TCP Handshake:** Browser initiates 3-way handshake (SYN → SYN-ACK → ACK) with the server IP on port 443.
3. **TLS Handshake:** Client Hello, Server Hello, certificate exchange, session key negotiation.
4. **HTTP Request:** GET request sent over the encrypted TLS tunnel.
5. **Server Processing:** Load balancer (like our Nginx ingress) receives request, routes to backend.
6. **Response:** HTML rendered by browser, triggering additional requests for CSS/JS/images.

In Kubernetes, this translates to: DNS resolves to Ingress IP → Nginx Ingress routes to Service → Service load balances to Pod → Pod responds."

---

### Q6. "What is the difference between TCP and UDP? When would you use each?"

**[YOUR ANSWER]:**
"**TCP** is connection-oriented — guarantees delivery, order, and error checking via the 3-way handshake. Used for web (HTTP/S), databases, SSH, anything where data integrity matters.

**UDP** is connectionless — fire and forget. Lower latency, no retransmission. Used for DNS, video streaming, VoIP, real-time gaming where a dropped packet is better than a delayed one.

In our Kubernetes setup, DNS (CoreDNS) uses UDP on port 53. Applications that need reliable service discovery use TCP. Monitoring agents like Datadog use TCP to ensure metrics are delivered."

---

### Q7. "What is a VLAN and how does it relate to VPC in cloud?"

**[YOUR ANSWER]:**
"A VLAN (Virtual LAN) is a logical segmentation of a physical network, creating isolated broadcast domains. Used in on-premise datacenters.

A VPC (Virtual Private Cloud) is the cloud equivalent — a logically isolated network in AWS or GCP. You control IP ranges, subnets, route tables, internet gateways, and NAT gateways.

At Altruist, I designed a VPC with public subnets for load balancers, private subnets for application servers, and isolated subnets for RDS databases. Security groups acted as virtual firewalls. This follows the defense-in-depth pattern — database is never exposed to the internet."

---

## SECTION 3: GIT & VERSION CONTROL

---

### Q8. "What is the difference between git merge and git rebase?"

**[YOUR ANSWER]:**
"Both integrate changes from one branch to another, but differently:

**git merge** creates a new merge commit that ties together the histories of two branches. History is preserved as-is, including all intermediate commits. Good for shared/public branches.

**git rebase** replays commits from one branch on top of another, creating a linear history. Makes the log cleaner. Good for feature branches before merging.

```bash
# Merge approach
git checkout main
git merge feature/new-service  # Creates merge commit

# Rebase approach
git checkout feature/new-service
git rebase main               # Replays commits on top of main
git checkout main
git merge feature/new-service  # Fast-forward merge
```

**Golden Rule:** Never rebase shared/public branches — it rewrites history and breaks others' clones."

---

### Q9. "What is a git cherry-pick? When would you use it?"

**[YOUR ANSWER]:**
"`git cherry-pick` applies a specific commit from one branch to another without merging the entire branch.

```bash
git cherry-pick abc1234  # Apply commit abc1234 to current branch
```

**Use case:** A critical hotfix was committed to a feature branch by mistake. You need to apply just that fix to main/production without merging the entire feature. I've used this at Vendasta when a security patch needed to go to production immediately, but the full feature wasn't ready."

---

## SECTION 4: SRE PRINCIPLES

---

### Q10. "What is the difference between SLI, SLO, SLA, and Error Budget?"

**[WHY THEY ASK]:** Core SRE concept — every SRE role will ask this.

**[YOUR ANSWER]:**
"These are the four pillars of reliability measurement:

**SLI (Service Level Indicator):** A specific, measurable metric that indicates service health.
- Example: '99th percentile latency for checkout API'
- Example: 'Percentage of successful HTTP 2xx responses'

**SLO (Service Level Objective):** A target value for an SLI, agreed internally between engineering and product.
- Example: 'Checkout API p99 latency must be < 500ms for 99.9% of requests in a 30-day window'

**SLA (Service Level Agreement):** A contractual commitment to customers, with financial penalties if breached.
- Example: 'We guarantee 99.5% uptime monthly. If we breach, customers get credits.'
- SLO is always stricter than SLA — internal target provides a buffer.

**Error Budget:** The acceptable amount of unreliability allowed before the SLO is breached.
- Formula: Error Budget = 1 - SLO
- For 99.9% SLO: Error budget = 0.1% = 43.8 minutes/month
- When budget is exhausted, new feature releases are halted until budget recovers.

**Real example from Vendasta:**
'We had a 99.9% SLO on our core API. One month, a bad deployment consumed 60% of our error budget in 2 hours. I flagged it to the team, we rolled back, wrote a post-mortem, and implemented canary deployments to prevent future budget blowouts.'"

---

### Q11. "What is toil in SRE, and how do you reduce it?"

**[YOUR ANSWER]:**
"Toil is manual, repetitive, automatable work that has no lasting value — it grows with traffic linearly.

Examples of toil:
- Manually restarting crashed pods
- Manually provisioning infrastructure for each new team
- Manually copying logs to S3 for analysis
- Running the same commands every deploy

**SRE principle:** Keep toil below 50% of engineering time. The rest should be engineering work that creates lasting value.

**How I reduce toil:**
- Automated self-healing with Kubernetes liveness probes (pod restarts itself)
- Terraform modules for self-service infrastructure provisioning
- ArgoCD for zero-touch GitOps deployments
- Runbooks converted into automated remediation playbooks

At Vendasta, I tracked our on-call toil by categorizing alert types in Datadog. 40% of pages were for a specific service restart. I automated the restart with a Kubernetes operator and eliminated that entire class of alerts."

---

### Q12. "What is the difference between monitoring, observability, and alerting?"

**[YOUR ANSWER]:**
"These are related but distinct concepts:

**Monitoring** is watching known failure modes — you instrument specific metrics and alert when thresholds breach. It answers: 'Is the system working?'

**Observability** is the property of a system that lets you understand its internal state from its external outputs — logs, metrics, traces. It answers: 'Why is the system behaving this way?'

**Alerting** is the mechanism that notifies on-call engineers when something requires human attention.

The three pillars of observability are:
- **Metrics** — quantitative measures over time (Prometheus, Datadog)
- **Logs** — event records (structured JSON logs, Datadog Log Management)
- **Traces** — request flow across services (Datadog APM, Jaeger)

At Vendasta, we had good monitoring but poor observability early on. When an incident happened, we knew a service was down but not why. I led the initiative to add structured logging and Datadog APM tracing, which reduced our mean-time-to-diagnose (MTTD) from 25 minutes to under 5 minutes."

---

### Q13. "How do you approach a post-mortem after a production incident?"

**[YOUR ANSWER]:**
"I follow a blameless post-mortem culture — the goal is to learn and improve, not blame individuals.

**Structure of a post-mortem:**
1. **Incident summary** — what happened, when, who was affected, duration
2. **Timeline** — exact sequence of events from first alert to resolution
3. **Root cause analysis** — 5 Whys technique to find the actual root cause
4. **Contributing factors** — what made the incident worse or harder to detect
5. **Impact** — customer impact, revenue impact, SLO budget consumed
6. **Action items** — concrete, assigned, time-bound tasks to prevent recurrence
7. **What went well** — what parts of the response worked correctly

**Real example:** After a GKE pod OOMKill incident at Vendasta:
- Root cause: memory leak in new library
- Action items: memory profiling in CI, add memory limits to all pods, canary deployments
- What went well: ArgoCD rollback was fast, on-call response was under SLA

I publish post-mortems to a shared Confluence space so the entire engineering org can learn."

---

### Q14. "What is chaos engineering and have you practiced it?"

**[YOUR ANSWER]:**
"Chaos engineering is the practice of intentionally introducing failures into production systems to identify weaknesses before they cause real incidents. Netflix pioneered this with Chaos Monkey.

**Principles:**
- Start with a steady state (normal behavior)
- Hypothesize what will happen when you inject failure
- Run experiments in production or a production-like environment
- Look for deviations from steady state — those are your weaknesses

**Tools:** Chaos Monkey, Gremlin, Litmus Chaos (for Kubernetes)

**At Vendasta:** I ran controlled chaos experiments using Litmus — terminating random pods to verify our liveness/readiness probes worked correctly, draining nodes to test PodDisruptionBudgets, and simulating network latency to verify circuit breakers. Each experiment led to at least one reliability improvement."

---

## SECTION 5: CI/CD FUNDAMENTALS

---

### Q15. "What is the difference between Continuous Integration, Continuous Delivery, and Continuous Deployment?"

**[YOUR ANSWER]:**
"These are three stages of the automated software delivery pipeline:

**Continuous Integration (CI):** Developers frequently merge code to a shared branch. Automated builds and tests run on every commit to catch integration issues early. Goal: merge small changes often.

**Continuous Delivery (CD):** The codebase is always in a deployable state. After CI passes, the build artifact is automatically deployed to staging or pre-production. Deployment to production requires a manual approval gate.

**Continuous Deployment:** Goes one step further — every change that passes all tests is automatically deployed to production without manual approval. Requires excellent test coverage and monitoring.

**At Vendasta:** We use Continuous Delivery — code merged to main auto-deploys to staging via ArgoCD. Production deployments require a manual approval in our GitHub Actions workflow to allow for human review. This balances speed with safety."

---

*Next: Move to File 03 — Kubernetes & Docker Deep Dive*
