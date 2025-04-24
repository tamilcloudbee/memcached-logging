# memcached-logging




### Create the Memcached stats logging script

```
sudo -i


sudo tee /var/lib/memcached-log-stats.sh > /dev/null << 'EOF'
#!/bin/bash

DATE=$(date '+%Y-%m-%d_%H-%M-%S')
INSTANCE_ID=$(curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/id)
ZONE=$(curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/zone | awk -F/ '{print $NF}')
MEMCACHED_IP="127.0.0.1"
MEMCACHED_PORT="11211"
LOG_FILE="/var/log/memcached_metrics.log"

# Fetch the memcached stats
stats_output=$( (echo "stats"; sleep 1) | nc -w 1 "$MEMCACHED_IP" "$MEMCACHED_PORT" 2>/dev/null )

# Function to extract a numeric value from the stats output
extract_stat() {
  echo "$stats_output" | grep "^STAT $1" | awk '{print $3}' | head -n 1
}

# Extract relevant metrics
current_connections=$(extract_stat "curr_connections")
get_hits=$(extract_stat "get_hits")
get_misses=$(extract_stat "get_misses")
set_hits=$(extract_stat "set_hits")
set_misses=$(extract_stat "set_misses")
bytes_used=$(extract_stat "bytes")

# Write the stats with timestamp to the log file and stdout
{
  echo "--- $DATE ---"
  echo "Instance ID: $INSTANCE_ID"
  echo "Zone: $ZONE"
  echo "Current Connections: $current_connections"
  echo "Get Hits: $get_hits"
  echo "Get Misses: $get_misses"
  echo "Set Hits: $set_hits"
  echo "Set Misses: $set_misses"
  echo "Bytes Used: $bytes_used"
  echo ""
} | tee -a "$LOG_FILE"
EOF

```

### Create the systemd service unit file
```

sudo tee /etc/systemd/system/memcached-log-stats.service > /dev/null << 'EOF'
[Unit]
Description=Log Memcached stats to file

[Service]
ExecStart=/bin/bash /var/lib/memcached-log-stats.sh
EOF



```

### Create the systemd timer unit file

```
sudo tee /etc/systemd/system/memcached-log-stats.timer > /dev/null << 'EOF'
[Unit]
Description=Run Memcached stats logger every hour

[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
EOF

```

### Reload systemd and start the timer

```
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now memcached-log-stats.timer


```

### Verify the output

```
cat /var/log/memcached_metrics.log
```
