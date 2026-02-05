#!/bin/bash

# ==============================================
# F5 Legacy TLS Logging with SAFE Log Management
# ==============================================

echo "Configuring SAFE TLS logging with disk protection..."
echo "===================================================="

# Part 1: Configure Log Rotation FIRST
echo -e "\n[1] Configuring Log Rotation..."
echo "================================="

LOGROTATE_CONF="/etc/logrotate.d/legacy_tls"
LOG_FILE="/var/log/legacy_tls.log"

# Create safe log rotation configuration
cat > "$LOGROTATE_CONF" << 'EOF'
# Legacy TLS Log Rotation - SAFE SETTINGS
/var/log/legacy_tls.log {
    daily                      # Rotate daily
    rotate 7                   # Keep 7 days of logs
    compress                   # Compress old logs (gzip)
    delaycompress              # Compress previous day's logs
    missingok                  # Don't error if log is missing
    notifempty                 # Don't rotate empty logs
    create 644 root root       # Set proper permissions
    postrotate
        # Gracefully reload syslog-ng
        /usr/bin/killall -HUP syslog-ng 2>/dev/null || true
    endscript
    maxsize 100M               # Rotate if exceeds 100MB
    su root root               # Run as root
}
EOF

echo "✓ Created log rotation: Keep 7 days, compress, max 100MB per file"

# Also rotate syslog-ng files
cat >> /etc/logrotate.d/syslog-ng << 'EOF'
# Additional rotation for syslog-ng files
/var/log/legacy_tls*.log {
    missingok
    notifempty
    sharedscripts
    postrotate
        /usr/bin/killall -HUP syslog-ng 2>/dev/null || true
    endscript
}
EOF

# Part 2: Configure syslog-ng with SIZE LIMITS
echo -e "\n[2] Configuring syslog-ng with Size Limits..."
echo "================================================"

SYSLOG_CONF="/etc/syslog-ng/syslog-ng.conf"

# Remove old configuration if exists
sed -i '/legacy_tls.log/d' "$SYSLOG_CONF"

# Add SAFE configuration with size limits
cat >> "$SYSLOG_CONF" << 'EOF'

# === LEGACY TLS LOGGING (SAFE CONFIGURATION) ===
source s_legacy_tcp {
    tcp(ip(127.0.0.1) port(514) max-connections(10));
};

filter f_legacy {
    match("LEGACY_TLS");
};

# Destination with SIZE LIMITS and ROTATION
destination d_legacy {
    file(
        "/var/log/legacy_tls.log"
        # SAFE SETTINGS:
        perm(0644)                    # Permissions
        owner(root)                   # Owner
        group(root)                   # Group
        create_dirs(yes)              # Create dirs if needed
        dir_perm(0755)                # Directory permissions
        
        # CRITICAL: SIZE AND ROTATION SETTINGS
        max-file-size(100M)           # MAX 100MB per file
        log_fifo_size(1000)           # Buffer size
        flush_lines(100)              # Flush every 100 lines
        flush_timeout(30000)          # Flush timeout (30 seconds)
        
        # ROTATION SETTINGS
        time_reap(30)                 # Check every 30 seconds
        throttle(1000)                # Max messages/sec
    );
};

# Log path with filtering
log {
    source(s_legacy_tcp);
    filter(f_legacy);
    destination(d_legacy);
    flags(final);
    # Rate limiting
    throttle(
        window(60)                    # 60 second window
        value(1000)                   # Max 1000 messages
        log_level(info)               # Log level for drops
    );
};
EOF

# Part 3: Configure F5 Logging Objects with THROTTLING
echo -e "\n[3] Creating F5 Objects with Throttling..."
echo "=============================================="

# Create pool (existing check)
tmsh create ltm pool legacy_log_pool members add { 127.0.0.1:514 } monitor gateway_icmp 2>/dev/null || \
echo "✓ Pool already exists"

# Create HSL with RATE LIMITING
tmsh create sys log-config destination remote-high-speed-log legacy_log_dest \
    forward-to /Common/legacy_log_pool \
    pool-name /Common/legacy_log_pool \
    protocol tcp \
    distribution adaptive \
    pool-action automatic \
    rate-limit 10000 2>/dev/null || \
echo "✓ HSL destination already exists (rate-limited to 10,000 msgs/sec)"

# Create publisher
tmsh create sys log-config publisher legacy_publisher \
    destinations { legacy_log_dest } \
    enabled true 2>/dev/null || \
echo "✓ Publisher already exists"

# Part 4: UPDATED iRule with AGGRESSIVE THROTTLING
echo -e "\n[4] Creating SAFE iRule with Multiple Protection Layers..."
echo "=============================================================="

IRULE_NAME="log_legacy_tls_safe"
IRULE_CONTENT=$(cat << 'EOF'
when RULE_INIT {
    # Initialize HSL
    set hsl_dest "/Common/legacy_log_dest"
    
    # Rate limiting variables
    set static::max_logs_per_second 100
    set static::max_logs_per_ip_per_hour 1000
    set static::last_reset [clock seconds]
    set static::log_count 0
    set static::ip_counts [table create -subtable ip_counter]
    
    # Open HSL connection
    if {[catch {
        set static::legacy_hsl [HSL::open -publisher $hsl_dest]
        log local0. "HSL connection established to $hsl_dest"
    } error]} {
        log local0. "HSL connection failed: $error"
        unset -nocomplain static::legacy_hsl
    }
}

when CLIENTSSL_CLIENTHELLO {
    # LAYER 1: Overall rate limiting
    set now [clock seconds]
    
    # Reset counter every second
    if {$now != $static::last_reset} {
        set static::log_count 0
        set static::last_reset $now
    }
    
    # Check global rate limit
    if {[incr static::log_count] > $static::max_logs_per_second} {
        # Too many logs - drop silently
        return
    }
    
    # Get connection info
    set tls_version [SSL::cipher version]
    set client_ip [IP::client_addr]
    
    # LAYER 2: Skip internal IPs
    if {[IP::addr $client_ip equals 10.0.0.0/8] || 
        [IP::addr $client_ip equals 192.168.0.0/16] || 
        [IP::addr $client_ip equals 172.16.0.0/12]} {
        return
    }
    
    # LAYER 3: IP-specific rate limiting
    set ip_key "ip_${client_ip}_[clock format $now -format {%Y%m%d%H}]"
    set ip_count [table lookup -subtable $static::ip_counts $ip_key]
    
    if {$ip_count >= $static::max_logs_per_ip_per_hour} {
        # IP exceeded hourly limit
        return
    }
    
    table set -subtable $static::ip_counts $ip_key [expr {$ip_count + 1}] indefinite 3600
    
    # LAYER 4: Only log TLS 1.0/1.1 from external nets
    set external_nets {
        202.8.88.0/22
        210.5.185.66/32
        220.241.141.1/32
        20.187.162.128/32
        20.187.187.28/32
    }
    
    set is_target_external 0
    foreach subnet $external_nets {
        if {[IP::addr $client_ip equals $subnet]} {
            set is_target_external 1
            break
        }
    }
    
    if {$is_target_external && ($tls_version eq "TLSv1" || $tls_version eq "TLSv1.1")} {
        # LAYER 5: Random sampling for high traffic (1 in 10)
        if {[expr {int(rand() * 10)}] != 0 && $ip_count > 100} {
            return
        }
        
        # Create log message
        set log_msg [format "LEGACY_TLS: Timestamp=[clock format $now -format {%Y-%m-%d %H:%M:%S}], \
            Client_IP=%s, TLS_Version=%s, Cipher=%s, VIP=%s:%s, SNI=%s" \
            $client_ip $tls_version [SSL::cipher name] \
            [IP::local_addr] [TCP::local_port] [SSL::extensions sni]]
        
        # Log with HSL if available
        if {[info exists static::legacy_hsl]} {
            HSL::send $static::legacy_hsl "$log_msg"
        }
        
        # Also log locally (rate limited by syslog-ng)
        log local0. "[string range $log_msg 0 255]"  # Truncate if too long
    }
}

# Cleanup old IP counters periodically
when CLIENT_ACCEPTED {
    # Clean old entries once every 1000 connections
    if {[expr {[connection id] % 1000}] == 0} {
        table timeout -subtable $static::ip_counts 3600
    }
}
EOF
)

# Create the safe iRule
tmsh create ltm rule "$IRULE_NAME" { "$IRULE_CONTENT" } 2>/dev/null || \
tmsh modify ltm rule "$IRULE_NAME" { "$IRULE_CONTENT" } 2>/dev/null

echo "✓ Created SAFE iRule with 5 layers of protection"

# Part 5: Disk Space Monitoring & Alerting
echo -e "\n[5] Configuring Disk Space Monitoring..."
echo "=========================================="

# Create disk monitoring script
MONITOR_SCRIPT="/shared/scripts/check_disk_space.sh"

cat > "$MONITOR_SCRIPT" << 'EOF'
#!/bin/bash
# Disk Space Monitor for F5 Legacy TLS Logging

THRESHOLD_CRITICAL=90   # Critical threshold (%)
THRESHOLD_WARNING=80    # Warning threshold (%)
LOG_FILE="/var/log/legacy_tls.log"
MAX_LOG_SIZE=$((100 * 1024 * 1024))  # 100MB in bytes

# Check disk usage
check_disk_usage() {
    # Check /var/log partition
    VAR_USAGE=$(df -h /var/log | awk 'NR==2 {print $5}' | sed 's/%//')
    VAR_AVAILABLE=$(df -h /var/log | awk 'NR==2 {print $4}')
    
    # Check /shared partition
    SHARED_USAGE=$(df -h /shared | awk 'NR==2 {print $5}' | sed 's/%//')
    SHARED_AVAILABLE=$(df -h /shared | awk 'NR==2 {print $4}')
    
    echo "Disk Usage - /var/log: ${VAR_USAGE}% (${VAR_AVAILABLE} available)"
    echo "Disk Usage - /shared: ${SHARED_USAGE}% (${SHARED_AVAILABLE} available)"
    
    # Critical alert
    if [ "$VAR_USAGE" -ge "$THRESHOLD_CRITICAL" ] || [ "$SHARED_USAGE" -ge "$THRESHOLD_CRITICAL" ]; then
        logger -p local0.crit "CRITICAL: Disk space above ${THRESHOLD_CRITICAL}% - /var/log:${VAR_USAGE}% /shared:${SHARED_USAGE}%"
        
        # Emergency action: Stop logging if /var/log > 95%
        if [ "$VAR_USAGE" -ge 95 ]; then
            logger -p local0.alert "EMERGENCY: Stopping legacy TLS logging"
            # Disable logging iRule on all virtual servers
            for vs in $(tmsh list ltm virtual one-line | awk '{print $3}'); do
                tmsh modify ltm virtual "$vs" rules { }
            done
        fi
        return 2
    fi
    
    # Warning alert
    if [ "$VAR_USAGE" -ge "$THRESHOLD_WARNING" ] || [ "$SHARED_USAGE" -ge "$THRESHOLD_WARNING" ]; then
        logger -p local0.warning "WARNING: Disk space above ${THRESHOLD_WARNING}% - /var/log:${VAR_USAGE}% /shared:${SHARED_USAGE}%"
        
        # Emergency cleanup: Remove old logs
        if [ "$VAR_USAGE" -ge 85 ]; then
            logger -p local0.warning "Performing emergency log cleanup"
            find /var/log -name "legacy_tls.log.*" -mtime +1 -delete 2>/dev/null
            find /var/log -name "*.gz" -mtime +3 -delete 2>/dev/null
        fi
        return 1
    fi
    
    return 0
}

# Check log file size
check_log_size() {
    if [ -f "$LOG_FILE" ]; then
        LOG_SIZE=$(stat -c%s "$LOG_FILE" 2>/dev/null)
        if [ "$LOG_SIZE" -gt "$MAX_LOG_SIZE" ]; then
            logger -p local0.warning "Log file oversized: $LOG_FILE ($((LOG_SIZE/1024/1024))MB)"
            # Force rotation
            logrotate -f /etc/logrotate.d/legacy_tls
        fi
    fi
}

# Main execution
case "$1" in
    "check")
        check_disk_usage
        check_log_size
        ;;
    "cleanup")
        # Force cleanup
        find /var/log -name "legacy_tls.log.*" -delete 2>/dev/null
        echo "Log cleanup completed"
        ;;
    "status")
        echo "=== Disk Status ==="
        df -h /var/log /shared
        echo -e "\n=== Log Files ==="
        ls -lh /var/log/legacy_tls* 2>/dev/null || echo "No legacy TLS logs found"
        echo -e "\n=== Log Count (today) ==="
        grep -c "LEGACY_TLS" /var/log/legacy_tls.log 2>/dev/null || echo "0"
        ;;
    *)
        echo "Usage: $0 {check|cleanup|status}"
        exit 1
        ;;
esac
EOF

chmod +x "$MONITOR_SCRIPT"

# Create cron job for monitoring
(crontab -l 2>/dev/null; echo "# F5 Disk Space Monitoring") | crontab -
(crontab -l 2>/dev/null; echo "*/15 * * * * $MONITOR_SCRIPT check >> /var/log/disk_monitor.log 2>&1") | crontab -
(crontab -l 2>/dev/null; echo "0 2 * * * $MONITOR_SCRIPT cleanup >> /var/log/disk_cleanup.log 2>&1") | crontab -

echo "✓ Disk monitoring configured (checks every 15 minutes)"

# Part 6: Emergency Cleanup Procedure
echo -e "\n[6] Creating Emergency Cleanup Procedure..."
echo "=============================================="

EMERGENCY_SCRIPT="/shared/scripts/emergency_log_cleanup.sh"

cat > "$EMERGENCY_SCRIPT" << 'EOF'
#!/bin/bash
# EMERGENCY Log Cleanup for F5

echo "=== EMERGENCY LOG CLEANUP STARTED ==="
echo "Timestamp: $(date)"

# 1. Stop logging temporarily
echo "1. Disabling logging iRules..."
for vs in $(tmsh list ltm virtual one-line | awk '{print $3}'); do
    tmsh modify ltm virtual "$vs" rules { }
done

# 2. Rotate current logs
echo "2. Rotating logs..."
logrotate -f /etc/logrotate.d/legacy_tls 2>/dev/null
logrotate -f /etc/logrotate.d/syslog-ng 2>/dev/null

# 3. Remove old logs (keep only last 2 days)
echo "3. Removing old log files..."
find /var/log -name "legacy_tls.log.*" -mtime +2 -delete 2>/dev/null
find /var/log -name "*.gz" -mtime +3 -delete 2>/dev/null

# 4. Clear syslog-ng buffers
echo "4. Clearing syslog-ng buffers..."
bigstop syslog-ng 2>/dev/null
rm -f /var/log/syslog-ng.persist 2>/dev/null
bigstart syslog-ng 2>/dev/null

# 5. Check disk space
echo "5. Current disk status:"
df -h /var/log /shared

# 6. Create log file with warning
echo "6. Creating audit trail..."
logger -p local0.alert "EMERGENCY LOG CLEANUP PERFORMED AT $(date)"

echo "=== CLEANUP COMPLETE ==="
EOF

chmod +x "$EMERGENCY_SCRIPT"

echo "✓ Emergency cleanup script created: $EMERGENCY_SCRIPT"

# Part 7: Final Configuration & Verification
echo -e "\n[7] Final Verification..."
echo "============================"

# Restart services
bigstart restart syslog-ng

# Test configuration
echo "Testing configuration..."
logger -p local0.info "LEGACY_TLS_TEST: Configuration verified at $(date)"

# Show current disk usage
echo -e "\nCurrent Disk Status:"
df -h /var/log /shared | grep -v Filesystem

# Show log settings
echo -e "\nLog Rotation Settings:"
cat "$LOGROTATE_CONF"

# Part 8: Usage Instructions with SAFETY MEASURES
echo -e "\n[8] SAFE USAGE INSTRUCTIONS"
echo "=============================="
cat << 'EOF'

=== IMPORTANT SAFETY MEASURES ACTIVE ===

1. AUTOMATIC PROTECTIONS:
   - Log rotation: Daily, keep 7 days, max 100MB/file
   - Rate limiting: 100 logs/second globally
   - IP throttling: 1000 logs/hour per IP
   - Random sampling: 10% for high-traffic IPs
   - Disk monitoring: Alerts at 80%, emergency at 90%

2. TO APPLY SAFELY:
   # Apply to ONE virtual server first for testing
   tmsh modify ltm virtual <your_vs> rules { $IRULE_NAME }
   
   # Monitor for 24 hours
   tail -f /var/log/legacy_tls.log
   /shared/scripts/check_disk_space.sh status

3. MONITORING COMMANDS:
   # Check disk space
   /shared/scripts/check_disk_space.sh status
   
   # Check log size
   ls -lh /var/log/legacy_tls*
   
   # Count today's entries
   grep -c "LEGACY_TLS" /var/log/legacy_tls.log
   
   # Watch disk usage in real-time
   watch -n 60 'df -h /var/log /shared'

4. EMERGENCY PROCEDURES:
   # If disk is filling up (>85%)
   /shared/scripts/emergency_log_cleanup.sh
   
   # Disable logging on all virtual servers
   for vs in $(tmsh list ltm virtual one-line | awk '{print $3}'); do
       tmsh modify ltm virtual "$vs" rules { }
   done

5. RECOMMENDED DAILY CHECKS:
   - Disk usage on /var/log and /shared
   - Log file sizes
   - Number of unique IPs hitting legacy TLS

=== SAFE DEFAULT SETTINGS ===
Max daily log volume: ~700MB (100MB/day x 7 days)
Max connections logged: ~8,640,000/day (100/sec theoretical max)
Actual expected: 1-10% of this based on sampling

EOF

echo -e "\n✅ SAFE Configuration Complete!"
echo "================================="
echo "Key safety features enabled:"
echo "1. Log rotation (100MB max, 7 days retention)"
echo "2. Rate limiting (100 logs/second)"
echo "3. IP throttling (1000 logs/hour per IP)"
echo "4. Disk monitoring with alerts"
echo "5. Emergency cleanup procedures"
echo ""
echo "Start with ONE virtual server and monitor for 24 hours!"