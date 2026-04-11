## `create_10_issues.sh` – Issue Generator Script

```bash
#!/bin/bash
# Creates 10 synthetic Linux issues for training.
# Usage: sudo ./create_10_issues.sh [--cleanup]

set -euo pipefail

# ------------------------------
# Configuration
# ------------------------------
TEMP_DIR="/tmp/devops_training"
ISSUE_DIR="$TEMP_DIR/issues"
LOG_FILE="$ISSUE_DIR/creation.log"
PORT=9999
ZOMBIE_BIN="$ISSUE_DIR/zombie_maker"
FAKE_SCRIPT="/usr/local/bin/mytool"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

# ------------------------------
# Cleanup function
# ------------------------------
cleanup() {
    echo -e "${GREEN}[*] Cleaning up all issues...${NC}"
    
    # 1. Kill CPU loop
    pkill -f "while.*bc -l" 2>/dev/null || true
    
    # 2. Remove large file
    rm -f "$ISSUE_DIR/bigfile"
    
    # 3. Stop & remove broken service
    systemctl stop broken@devops 2>/dev/null || true
    systemctl disable broken@devops 2>/dev/null || true
    rm -f /etc/systemd/system/broken@devops.service
    systemctl daemon-reload
    
    # 4. Restore /etc/hosts permissions
    chmod 644 /etc/hosts 2>/dev/null || true
    
    # 5. Resume and kill hung process
    if [[ -f "$ISSUE_DIR/hung.pid" ]]; then
        pid=$(cat "$ISSUE_DIR/hung.pid")
        kill -CONT "$pid" 2>/dev/null || true
        kill -9 "$pid" 2>/dev/null || true
        rm -f "$ISSUE_DIR/hung.pid"
    fi
    
    # 6. Remove cron job
    crontab -l 2>/dev/null | grep -v "Cron spam from devops training" | crontab - 2>/dev/null || true
    rm -f /tmp/cron_spam.log
    
    # 7. Kill zombie parent process
    pkill -f "./zombie_maker" 2>/dev/null || true
    rm -f "$ZOMBIE_BIN"
    
    # 8. Kill port conflict listeners
    pkill -f "nc -l -p $PORT" 2>/dev/null || true
    pkill -f "python3 -m http.server $PORT" 2>/dev/null || true
    
    # 9. Restore original /etc/resolv.conf (remove bad entry)
    if grep -q "192.0.2.1" /etc/resolv.conf; then
        sed -i '/nameserver 192.0.2.1/d' /etc/resolv.conf
    fi
    
    # 10. Remove fake script and restore PATH (if changed)
    rm -f "$FAKE_SCRIPT"
    # Ensure PATH is normal (no action needed, just cleanup)
    
    rm -rf "$TEMP_DIR"
    echo -e "${GREEN}[+] Cleanup finished.${NC}"
    exit 0
}

# ------------------------------
# Issue creation functions
# ------------------------------
issue1_high_cpu() {
    echo -e "${RED}[1] High CPU usage (infinite loop)${NC}"
    ( while :; do echo "scale=5000; a(1)*4" | bc -l >/dev/null 2>&1; done ) &
    renice -n 19 -p $! >/dev/null 2>&1
    echo "    CPU hog PID: $!"
}

issue2_full_disk() {
    echo -e "${RED}[2] Disk filled with 500MB dummy file${NC}"
    dd if=/dev/zero of="$ISSUE_DIR/bigfile" bs=1M count=500 status=none
}

issue3_broken_service() {
    echo -e "${RED}[3] Systemd service that fails to start${NC}"
    cat <<EOF > /etc/systemd/system/broken@devops.service
[Unit]
Description=Broken DevOps Service

[Service]
ExecStart=/bin/false
Restart=no
User=nobody

[Install]
WantedBy=multi-user.target
EOF
    systemctl daemon-reload
    systemctl enable broken@devops &>/dev/null || true
}

issue4_wrong_permissions() {
    echo -e "${RED}[4] World‑writable /etc/hosts${NC}"
    chmod 777 /etc/hosts
}

issue5_hung_process() {
    echo -e "${RED}[5] Stopped (hung) process (state T)${NC}"
    sleep 3600 &
    pid=$!
    kill -STOP "$pid"
    echo "$pid" > "$ISSUE_DIR/hung.pid"
}

issue6_cron_spam() {
    echo -e "${RED}[6] Cron job writing to /tmp every minute${NC}"
    (crontab -l 2>/dev/null; echo "* * * * * echo 'Cron spam from devops training' >> /tmp/cron_spam.log") | crontab -
}

issue7_zombie_process() {
    echo -e "${RED}[7] Zombie process (defunct)${NC}"
    # Write a small C program that forks and exits child without waiting
    cat <<'EOF' > "$ZOMBIE_BIN.c"
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
int main() {
    if (fork() == 0) {
        exit(0);  // child exits, parent doesn't wait -> zombie
    }
    sleep(300);   // parent keeps running
    return 0;
}
EOF
    gcc "$ZOMBIE_BIN.c" -o "$ZOMBIE_BIN" 2>/dev/null || { echo "    gcc not found, skipping zombie issue"; return; }
    "$ZOMBIE_BIN" &
    echo "    Zombie parent PID: $!"
}

issue8_port_conflict() {
    echo -e "${RED}[8] Port $PORT already in use (conflict)${NC}"
    # Start a netcat listener on $PORT
    nc -l -p "$PORT" >/dev/null 2>&1 &
    echo "    Listener started on port $PORT (PID: $!)"
    # Now a second service will fail to bind – we'll simulate by trying to start python http server
    # The student will need to find and kill the conflicting process.
    echo "    (Simulated conflict: try to start 'python3 -m http.server $PORT' – it will fail)"
}

issue9_dns_issue() {
    echo -e "${RED}[9] Broken DNS resolution (bad nameserver)${NC}"
    # Add a non‑responsive nameserver to /etc/resolv.conf
    echo "nameserver 192.0.2.1" >> /etc/resolv.conf
    echo "    Added bogus nameserver. 'ping google.com' will hang/timeout."
}

issue10_missing_executable() {
    echo -e "${RED}[10] Missing execute permission on /usr/local/bin/mytool${NC}"
    cat <<'EOF' > "$FAKE_SCRIPT"
#!/bin/bash
echo "This tool works now!"
EOF
    chmod 644 "$FAKE_SCRIPT"   # no execute bit
    echo "    Script created but not executable. Try: mytool"
}

# ------------------------------
# Main
# ------------------------------
if [[ "$1" == "--cleanup" ]]; then
    cleanup
fi

if [[ $EUID -ne 0 ]]; then
    echo "Please run as root (sudo)."
    exit 1
fi

mkdir -p "$ISSUE_DIR"

echo "============================================"
echo "  Linux Troubleshooting – 10 Real Issues"
echo "  Run with --cleanup to revert all"
echo "============================================"
echo ""

issue1_high_cpu
issue2_full_disk
issue3_broken_service
issue4_wrong_permissions
issue5_hung_process
issue6_cron_spam
issue7_zombie_process
issue8_port_conflict
issue9_dns_issue
issue10_missing_executable

echo ""
echo -e "${GREEN}[✓] All 10 issues created.${NC}"
echo "Give the trainee the task list below."
```

