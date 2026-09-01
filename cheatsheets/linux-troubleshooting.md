# Linux Troubleshooting & Diagnostic Commands Cheat Sheet

---

## ⚡ 1. CPU & Thread Profiling

```bash
# 1. View top processes sorted by CPU
top -b -n 1 | head -n 20

# 2. View individual CPU-consuming threads within a JVM process
top -H -p <pid>

# 3. Capture on-demand JVM thread dump using jcmd
jcmd <pid> Thread.print > /tmp/threaddump.txt

# 4. Generate async-profiler CPU FlameGraph (60-second profile)
asprof -d 60 -f /tmp/flamegraph.html <pid>
```

---

## ⚡ 2. Memory & Disk I/O Diagnostics

```bash
# 1. Check system virtual memory, swap, and context switches every 1 second
vmstat 1 10

# 2. Check disk I/O utilization & queue wait times per NVMe device
iostat -xz 1 10
# (Look for %util > 90% or await > 10ms)

# 3. Check disk space saturation
df -h
du -sh /var/log/* | sort -hr | head -n 10

# 4. Check Linux OOM Killer logs in kernel ring buffer
dmesg -T | grep -i "oom-killer"
```

---

## ⚡ 3. Networking & Socket Triage

```bash
# 1. Check listening ports and established TCP socket connections
ss -tulnp
ss -s # Socket summary: TIME-WAIT, CLOSE-WAIT counts

# 2. Capture TCP traffic on Port 8080 (Inspect HTTP headers)
tcpdump -i any -nnvv -s0 tcp port 8080 -A

# 3. Check DNS resolution latency and nameserver responses
dig +trace +stats api.internal
```
