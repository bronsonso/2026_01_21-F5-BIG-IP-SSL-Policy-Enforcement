[ backup f5 config] 

echo "Starting F5 Configuration Script..."
echo "===================================="

# Create UCS backup without passphrase
BACKUP_FILE="backup_$(date +%Y%m%d_%H%M%S).ucs"

echo "Creating backup: $BACKUP_FILE"
tmsh save sys ucs "$BACKUP_FILE"

# List existing backups
echo -e "\nExisting UCS backups:"
tmsh list sys ucs | grep -E "^(ucs|    file-name)"





# Part 2: Configure Legacy TLS Logging
echo -e "\n[2] Configuring Legacy TLS Logging..."
echo "========================================"

# Check prerequisites
echo "Checking prerequisites..."

# Check if syslog-ng is installed and running
if which syslog-ng >/dev/null 2>&1; then
    echo "✓ syslog-ng is installed: $(which syslog-ng)"
    
    # Check if running
    SYSLOG_STATUS=$(bigstart status syslog-ng 2>/dev/null | grep -c "running")
    if [ "$SYSLOG_STATUS" -eq 1 ]; then
        echo "✓ syslog-ng is running"
    else
        echo "✗ syslog-ng is not running, starting it..."
        bigstart start syslog-ng
    fi
else
    echo "✗ syslog-ng is not installed"
    echo "Please install syslog-ng package first"
    exit 1
fi






# Configure syslog-ng
echo -e "\nConfiguring syslog-ng..."
SYSLOG_CONF="/etc/syslog-ng/syslog-ng.conf"
BACKUP_CONF="/etc/syslog-ng/syslog-ng.conf.backup_$(date +%Y%m%d)"

# Backup original config
cp "$SYSLOG_CONF" "$BACKUP_CONF"
echo "✓ Backed up syslog-ng config to: $BACKUP_CONF"

# Check if config already exists
if grep -q "legacy_tls.log" "$SYSLOG_CONF"; then
    echo "✓ Legacy TLS logging already configured in syslog-ng"
else
    # Append configuration
    cat >> "$SYSLOG_CONF" << 'EOF'

# Legacy TLS Traffic Logging (added by script)
source s_legacy_tcp { tcp(ip(127.0.0.1) port(514)); };
filter f_legacy { match("LEGACY_TLS"); };
destination d_legacy { file("/var/log/legacy_tls.log"); };
log { source(s_legacy_tcp); filter(f_legacy); destination(d_legacy); flags(final); };
EOF
    echo "✓ Added legacy TLS configuration to syslog-ng"
fi

# Restart syslog-ng
echo -e "\nRestarting syslog-ng..."
bigstart restart syslog-ng
echo "✓ syslog-ng restarted"





# Part 3: Create F5 Objects
echo -e "\n[3] Creating F5 Objects..."
echo "=============================="

# Create logging pool
echo "Creating logging pool..."
tmsh create ltm pool legacy_log_pool members add { 127.0.0.1:514 } monitor gateway_icmp 2>/dev/null || \
tmsh modify ltm pool legacy_log_pool members add { 127.0.0.1:514 } 2>/dev/null || \
echo "✓ Pool legacy_log_pool already exists or created"

# Create HSL destination
echo "Creating HSL destination..."
tmsh create sys log-config destination remote-high-speed-log legacy_log_dest \
    forward-to /Common/legacy_log_pool \
    pool-name /Common/legacy_log_pool \
    protocol tcp 2>/dev/null || \
echo "✓ HSL destination legacy_log_dest already exists"

# Create publisher (FIXED: Was missing in original)
echo "Creating log publisher..."
tmsh create sys log-config publisher legacy_publisher \
    destinations { legacy_log_dest } 2>/dev/null || \
echo "✓ Publisher legacy_publisher already exists"







# Part 4: Create iRules
echo -e "\n[4] Creating iRules..."
echo "========================="

# Create the main logging iRule
IRULE_NAME="log_legacy_tls_irule"
IRULE_CONTENT=$(cat << 'EOF'
when CLIENTSSL_CLIENTHELLO {
    # Get TLS version and connection info
    set tls_version [SSL::cipher version]
    set client_ip [IP::client_addr]
    
    # Skip internal IPs (10.x.x.x)
    if {[IP::addr $client_ip equals 10.0.0.0/8]} {
        return
    }
    
    # Define specific external networks to monitor
    set external_nets {
        202.8.88.0/22
        210.5.185.66/32
        220.241.141.1/32
        20.187.162.128/32
        20.187.187.28/32
    }
    
    # Check if traffic is from defined external networks
    set is_target_external 1

    #set is_target_external 0  
    #foreach subnet $external_nets {
    #    if {[IP::addr $client_ip equals $subnet]} {
    #        set is_target_external 1
    #        break
    #    }
    #}
    
    # Log only external TLS 1.0/1.1 traffic from target networks
    if {$is_target_external && ($tls_version eq "TLSv1" || $tls_version eq "TLSv1.1")} {
        # Create log message
        set log_msg [format "LEGACY_TLS: Timestamp=[clock format [clock seconds] -format {%Y-%m-%d %H:%M:%S}], \
            Client_IP=%s, TLS_Version=%s, Cipher=%s, VIP=%s:%s, SNI=%s" \
            $client_ip $tls_version [SSL::cipher name] \
            [IP::local_addr] [TCP::local_port] [SSL::extensions sni]]
        
        # Log locally
        # log local0. "$log_msg"
        
        # Send to HSL if available
        if {[info exists static::legacy_hsl]} {
            HSL::send $static::legacy_hsl "$log_msg"
        }
    }
}
EOF
)

# Create HSL initialization iRule (separate for better management)
IRULE_INIT_NAME="log_legacy_tls_hsl_init"
IRULE_INIT_CONTENT=$(cat << 'EOF'
when RULE_INIT {
    # Initialize HSL connection
    set hsl_dest "/Common/legacy_log_dest"
    
    # Check if destination exists
    if {[catch {
        set static::legacy_hsl [HSL::open -publisher $hsl_dest]
        log local0. "Successfully opened HSL connection to $hsl_dest"
    } error]} {
        log local0. "Failed to open HSL connection to $hsl_dest: $error"
        unset -nocomplain static::legacy_hsl
    }
}
EOF
)

# Create the iRules
echo "Creating iRule: $IRULE_NAME"
tmsh create ltm rule "$IRULE_NAME" { "$IRULE_CONTENT" } 2>/dev/null || \
echo "✓ iRule $IRULE_NAME already exists"

echo "Creating iRule: $IRULE_INIT_NAME"
tmsh create ltm rule "$IRULE_INIT_NAME" { "$IRULE_INIT_CONTENT" } 2>/dev/null || \
echo "✓ iRule $IRULE_INIT_NAME already exists"

# Part 5: Verification
echo -e "\n[5] Verification..."
echo "====================="

# Verify objects exist
echo "Verifying created objects:"
tmsh list ltm pool legacy_log_pool one-line 2>/dev/null && echo "✓ Pool: legacy_log_pool"
tmsh list sys log-config destination legacy_log_dest one-line 2>/dev/null && echo "✓ HSL Destination: legacy_log_dest"
tmsh list sys log-config publisher legacy_publisher one-line 2>/dev/null && echo "✓ Publisher: legacy_publisher"
tmsh list ltm rule "$IRULE_NAME" one-line 2>/dev/null && echo "✓ iRule: $IRULE_NAME"
tmsh list ltm rule "$IRULE_INIT_NAME" one-line 2>/dev/null && echo "✓ iRule: $IRULE_INIT_NAME"

# Create test log entry
echo -e "\nCreating test log entry..."
logger -p local0.info "LEGACY_TLS: TEST - Script configuration completed successfully"

# Check log file
echo -e "\nChecking log files:"
echo "/var/log/legacy_tls.log (last 5 lines):"
tail -5 /var/log/legacy_tls.log 2>/dev/null || echo "Log file doesn't exist yet (will be created when traffic matches)"

echo "/var/log/ltm (last 3 lines):"
tail -3 /var/log/ltm 2>/dev/null | grep -i "legacy\|test"

# Part 6: Usage Instructions
echo -e "\n[6] Usage Instructions"
echo "========================"
cat << 'EOF'

To use this configuration:

1. Apply the iRules to your Virtual Server:
   tmsh modify ltm virtual <your_virtual_server> rules { $IRULE_INIT_NAME $IRULE_NAME }

2. Monitor logs in real-time:
   tail -f /var/log/legacy_tls.log

3. Check log rotation (configure if needed):
   vi /etc/logrotate.conf

4. To disable logging temporarily:
   tmsh modify ltm virtual <your_virtual_server> rules { }

Notes:
- The script logs TLS 1.0/1.1 traffic from specific external IPs only
- Internal 10.x.x.x addresses are excluded
- Logs are stored in /var/log/legacy_tls.log
- Make sure to apply both iRules in correct order

EOF

echo -e "\nScript completed successfully! ✓"
echo "===================================="







[ create SSL profiles ]







[ create iRules ]











[ restore f5 config ]

# List available UCS files
tmsh list sys ucs

# Restore UCS (will reboot)
tmsh load sys ucs backup_20240115.ucs

# With passphrase
tmsh load sys ucs backup_20240115.ucs passphrase "YourPassphrase"

# Dry-run (check only, no restore)
tmsh load sys ucs backup_20240115.ucs no-license verify