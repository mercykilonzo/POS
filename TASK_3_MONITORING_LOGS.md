# Task 3: Monitoring and Logs
## Deliverables: System Logs

---

## 3.1 How to Monitor Real-Time Logs (1 Hr)

### Overview

Real-time log monitoring allows you to observe transaction processing as it happens, diagnose issues, and track system performance.

### Log File Locations

#### **Main Application Logs**

**Location**: `/modules/jpts-app/build/install/jpts-app/`

```
jpts-app/
├── logs/
│   ├── jpts.log              (Primary log file)
│   ├── jpts.log.1            (Rotated logs)
│   ├── jpts.log.2
│   └── ...
├── var/
│   ├── q2.pid                (Process ID)
│   └── q2.fifo               (Named pipe for CLI)
└── bin/
    └── q2                    (Executable)
```

#### **Log Configuration**

**File**: `modules/jpts-app/src/main/resources/log4j.properties`

**Default Configuration**:
```properties
# Root logger
log4j.rootLogger=INFO, JPTS

# Appender: JPTS file
log4j.appender.JPTS=org.apache.log4j.RollingFileAppender
log4j.appender.JPTS.File=logs/jpts.log
log4j.appender.JPTS.MaxFileSize=10MB
log4j.appender.JPTS.MaxBackupIndex=10
log4j.appender.JPTS.layout=org.apache.log4j.PatternLayout
log4j.appender.JPTS.layout.ConversionPattern=%d [%t] %-5p %c - %m%n

# Logger levels
log4j.logger.org.jpos=DEBUG
log4j.logger.org.jpos.transaction=DEBUG
log4j.logger.org.jpos.gw=DEBUG
```

### Real-Time Log Monitoring Tools

#### **1. Using `tail -f` Command**

**Live Follow Mode**:
```bash
cd /modules/jpts-app/build/install/jpts-app
tail -f logs/jpts.log
```

**Output Example**:
```
2025-12-04 10:23:45,123 [Thread-1] DEBUG org.jpos.gw.Incoming - doPrepare: id=1234
2025-12-04 10:23:45,145 [Thread-1] DEBUG org.jpos.gw.Incoming - Incoming message converted
2025-12-04 10:23:45,167 [Thread-1] DEBUG org.jpos.gw.participant.TxnName - TXNNAME=0200.00
2025-12-04 10:23:45,189 [Thread-1] DEBUG org.jpos.gw.TranslatePin - PIN translation started
2025-12-04 10:23:45,201 [Thread-1] DEBUG org.jpos.gw.TranslatePin - PIN translated successfully
2025-12-04 10:23:45,223 [Thread-1] INFO org.jpos.jpts.CreateTranLog - Transaction 12345 created
```

#### **2. Filter Logs in Real-Time**

**Follow only ERROR logs**:
```bash
tail -f logs/jpts.log | grep ERROR
```

**Follow specific component**:
```bash
tail -f logs/jpts.log | grep "org.jpos.gw"
```

**Follow transaction creation**:
```bash
tail -f logs/jpts.log | grep -E "CreateTranLog|Transaction.*created"
```

**Follow routing decisions**:
```bash
tail -f logs/jpts.log | grep -E "TXNNAME|routing|destination"
```

#### **3. Multi-File Tailing**

**Monitor multiple services**:
```bash
tail -f /modules/jpts-app/build/install/jpts-app/logs/jpts.log \
        /modules/ss-wiseasy/logs/wiseasy.log \
        /modules/db-dev/logs/db.log
```

### Log Levels and What They Mean

| Level | When Used | Example |
|-------|-----------|---------|
| TRACE | Detailed function entry/exit | `Entering doPrepare()` |
| DEBUG | Detailed component flow | `Incoming message converted` |
| INFO | Significant events | `Transaction 12345 created` |
| WARN | Potential issues | `No matching route found` |
| ERROR | Error conditions | `Database connection failed` |
| FATAL | System failures | `Cannot start application` |

### Important Log Markers

#### **Transaction Lifecycle Logs**

```
[START] Incoming participant receives message
2025-12-04 10:23:45,123 [Thread-1] DEBUG org.jpos.gw.Incoming - doPrepare: id=1234

[ROUTING] TxnName derives transaction name
2025-12-04 10:23:45,145 [Thread-1] DEBUG org.jpos.gw.participant.TxnName - TXNNAME=0200.00

[PIN] PIN translation (if field 52 present)
2025-12-04 10:23:45,167 [Thread-1] DEBUG org.jpos.gw.TranslatePin - PIN translation...

[DATABASE] Transaction log created
2025-12-04 10:23:45,189 [Thread-1] INFO org.jpos.jpts.CreateTranLog - TranLog id=999

[SEND] Message sent to destination station
2025-12-04 10:23:45,201 [Thread-1] DEBUG org.jpos.transaction.participant.SendHost - Sending to DS

[RESPONSE] Response received and processed
2025-12-04 10:23:45,223 [Thread-1] DEBUG org.jpos.transaction.participant.ReceiveHost - Response received

[COMPLETE] Transaction completed
2025-12-04 10:23:45,245 [Thread-1] INFO org.jpos.transaction.TransactionManager - TX 1234 completed
```

#### **Error Condition Logs**

```
[ERROR] Connection failure
2025-12-04 10:23:50,000 [Thread-1] ERROR org.jpos.transaction.participant.SendHost - 
    Connection to ds-aci failed: Connection refused

[ERROR] Timeout
2025-12-04 10:23:55,000 [Thread-1] ERROR org.jpos.transaction.participant.ReceiveHost - 
    Timeout waiting for response from ds-aci

[ERROR] Database error
2025-12-04 10:24:00,000 [Thread-1] ERROR org.jpos.jpts.CreateTranLog - 
    Cannot insert transaction: Unique constraint violation
```

### Interactive Monitoring with jPTS Console

#### **Connect to Running Instance**

```bash
# From jPTS directory
./bin/q2 -c interactive
```

#### **Available Commands**

```
status              # Show system status
ps                  # Show active participants
txnstats            # Show transaction statistics
head -n 100         # Show recent logs
tail -n 50          # Show last 50 log lines
list                # Show configuration
help                # Show available commands
```

#### **Example Session**

```bash
$ ./bin/q2 -c interactive

q2> status
jPTS System Status:
  Uptime: 2 days 3 hours
  Transactions Processed: 15,234
  Success Rate: 98.7%
  
q2> txnstats
Transaction Statistics:
  Auth (0200): 10,234 (Success: 10,100)
  Settlement (0220): 5,000 (Success: 5,000)
  Reversal (0400): 0

q2> tail -n 20
[Last 20 log lines]
```

### Log Analysis Patterns

#### **Pattern 1: Transaction Success Flow**

```bash
# Search for complete successful transaction
grep "id=1234" logs/jpts.log | head -20

# Output should show:
# 1. Incoming message received
# 2. TXNNAME derived
# 3. PIN translated (if present)
# 4. TranLog created
# 5. Send to destination
# 6. Response received
# 7. Transaction completed
```

#### **Pattern 2: Find Failed Transactions**

```bash
# Find transactions that failed
grep "ERROR\|FAILED\|Exception" logs/jpts.log

# Get context (5 lines before/after)
grep -B5 -A5 "ERROR" logs/jpts.log
```

#### **Pattern 3: PIN Translation Issues**

```bash
# Find PIN translation attempts
grep "TranslatePin" logs/jpts.log

# Expected flow:
# - PIN validation
# - Source key retrieval
# - Destination key retrieval
# - Translation execution
# - Success/failure status
```

#### **Pattern 4: Routing Decisions**

```bash
# Trace routing decisions
grep -E "TXNNAME|routing|destination|SelectGroup" logs/jpts.log

# Example output:
# TXNNAME=0200.00
# Route selected: ds-aci
# Sending to: aci-processor
```

### Performance Monitoring

#### **Transaction Processing Time**

```bash
# Extract transaction start and end times
grep "doPrepare.*id=1234\|completed.*id=1234" logs/jpts.log

# Calculate manually or use AWK
awk '/doPrepare.*id=1234/ {start=$1; next} 
     /completed.*id=1234/ {end=$1; 
     print "Duration: " (end - start) "ms"}' logs/jpts.log
```

#### **Bottleneck Identification**

```bash
# Find slowest participants
grep "doPrepare" logs/jpts.log | \
  awk -F'[: ]' '{
    class=$NF; 
    getline next; 
    if (next ~ /ERROR|Exception/) return; 
    print class
  }' | sort | uniq -c | sort -rn
```

### Advanced Monitoring with Log Aggregation

#### **Using Log Aggregation Tools**

If available, integrate with:
- **ELK Stack** (Elasticsearch, Logstash, Kibana)
- **Splunk**
- **Graylog**

**Basic Logstash Configuration**:
```
input {
  file {
    path => "/modules/jpts-app/build/install/jpts-app/logs/jpts.log"
    start_position => "beginning"
  }
}

filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} \[%{DATA:thread}\] %{LOGLEVEL:level} %{JAVACLASS:class} - %{GREEDYDATA:msg}" }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "jpts-%{+YYYY.MM.dd}"
  }
}
```

---

## 3.2 How to Retrieve and Read Archived Logs (1 Hr)

### Archived Log Structure

#### **Log Rotation Configuration**

**From log4j.properties**:
```properties
log4j.appender.JPTS.MaxFileSize=10MB
log4j.appender.JPTS.MaxBackupIndex=10
```

**Behavior**:
- When `jpts.log` reaches 10MB, it's renamed to `jpts.log.1`
- Previous backups shift: `log.1` → `log.2`, `log.2` → `log.3`, etc.
- Maximum 10 archived files retained
- Oldest file deleted when 11th rotation would occur

#### **Archive Locations**

```
jpts-app/logs/
├── jpts.log          (Current - up to 10MB)
├── jpts.log.1        (Most recent archive)
├── jpts.log.2
├── ...
└── jpts.log.10       (Oldest archive retained)
```

### Retrieving Archived Logs

#### **Method 1: List All Log Files**

```bash
cd /modules/jpts-app/build/install/jpts-app/logs
ls -lh jpts.log*
```

**Output**:
```
-rw-r--r-- 1 user group 8.5M Dec  4 10:30 jpts.log
-rw-r--r-- 1 user group 10M  Dec  4 10:15 jpts.log.1
-rw-r--r-- 1 user group 10M  Dec  4 09:45 jpts.log.2
-rw-r--r-- 1 user group 10M  Dec  3 23:30 jpts.log.3
-rw-r--r-- 1 user group 10M  Dec  3 15:00 jpts.log.4
```

#### **Method 2: View Specific Archive**

```bash
# View oldest archive
less jpts.log.10

# View most recent archive
less jpts.log.1

# Search within archive
grep "ERROR" jpts.log.5

# Count errors in archive
grep -c "ERROR" jpts.log.3
```

#### **Method 3: Combine Multiple Archives**

```bash
# Search across all archives for specific transaction
grep "id=12345" jpts.log*

# Example output:
# jpts.log:2025-12-04 10:23:45,123 [Thread-1] DEBUG ... id=12345 ...
# jpts.log.1:2025-12-04 10:23:46,456 [Thread-1] DEBUG ... id=12345 ...
```

### Analyzing Archived Logs

#### **Analysis 1: Historical Transaction Search**

**Scenario**: Find a transaction from yesterday

```bash
# Check which archive has yesterday's logs
ls -l jpts.log.* | grep "Dec  3"

# Search in that archive
grep "2025-12-03" jpts.log.4 | grep "id=5678"

# View full transaction flow
grep -E "2025-12-03.*id=5678|id=5678.*2025-12-03" jpts.log.4
```

#### **Analysis 2: Error Frequency Over Time**

```bash
# Count errors per archive
for file in jpts.log.{1..10}; do
  if [ -f "$file" ]; then
    count=$(grep -c "ERROR" "$file" 2>/dev/null || echo 0)
    echo "$file: $count errors"
  fi
done
```

**Output**:
```
jpts.log.1: 45 errors (Most recent)
jpts.log.2: 32 errors
jpts.log.3: 18 errors
jpts.log.4: 25 errors
jpts.log.5: 12 errors
... (older logs)
```

#### **Analysis 3: Extract Transaction Statistics**

```bash
# Extract all successful transactions from archived logs
grep -h "Transaction.*created\|completed" jpts.log.* | \
  awk '{print $1, $2}' | \
  sort | uniq -c

# Count transactions per hour
grep "INFO.*CreateTranLog" jpts.log.* | \
  cut -d' ' -f2 | cut -d: -f1 | uniq -c
```

#### **Analysis 4: PIN Translation Success Rate**

```bash
# Count PIN translation attempts
attempts=$(grep -c "PIN translation" jpts.log.*)
successes=$(grep -c "PIN.*success" jpts.log.*)
failures=$(grep -c "PIN.*fail" jpts.log.*)

echo "PIN Translations: $attempts attempts, $successes success, $failures failures"
```

### Archiving Strategy

#### **Long-Term Archive (Compressed)**

```bash
# Compress old archives to save space
gzip jpts.log.10

# View compressed logs
zcat jpts.log.10.gz | grep "ERROR"

# Search in compressed file
zgrep "id=12345" jpts.log.*.gz
```

#### **Archive to External Storage**

```bash
# Copy archives to external location (daily)
cp jpts.log.10 /backup/jpts-logs/$(date +%Y-%m-%d)-jpts.log.10
gzip /backup/jpts-logs/$(date +%Y-%m-%d)-jpts.log.10

# Or use rsync for incremental backup
rsync -av logs/ /backup/jpts-logs/
```

#### **Create Archive Index**

```bash
# Generate index of all log files with dates
ls -l jpts.log.* | awk '{
  print $6, $7, $8, $9, "- Lines:", $(system("wc -l "$9" 2>/dev/null"))
}' > ../log_index.txt

# Result shows which archive contains which date
```

### Extracting Data from Archives

#### **Database Integration**

Log entries can be parsed and stored in database for querying:

```bash
# Extract transaction logs to CSV
grep "CreateTranLog" jpts.log.* | \
  awk -F'[: ]' '{
    print $1","$2","$NF
  }' > transactions.csv

# Import to database
psql -U jpos -d flocash_jpts -c "
  COPY transaction_logs FROM 'transactions.csv' 
  WITH (FORMAT CSV, DELIMITER ',');"
```

### Audit Trail Management

#### **Regulatory Compliance**

For financial compliance (PCI-DSS, etc.):

```bash
# Retain logs for minimum period (typically 1-3 years)
# Copy to WORM (Write Once Read Many) storage

# Generate audit report
echo "=== Audit Report $(date) ===" > audit_report.txt
echo "Total Transactions:" $(grep -h "CreateTranLog" jpts.log* | wc -l) >> audit_report.txt
echo "Errors:" $(grep -ch "ERROR" jpts.log* | wc -l) >> audit_report.txt
echo "Log Files:" >> audit_report.txt
ls -lh jpts.log* >> audit_report.txt
```

#### **Secure Deletion**

```bash
# Securely delete logs after retention period
# Using shred for cryptographic deletion
shred -vfz -n 3 jpts.log.10

# Or use rm with secure deletion enabled (system-dependent)
rm -P jpts.log.10
```

### Log Rotation Troubleshooting

#### **Issue: Logs Not Rotating**

**Check log4j configuration**:
```bash
grep -A3 "RollingFileAppender" modules/jpts-app/src/main/resources/log4j.properties
```

**Verify file permissions**:
```bash
ls -la logs/ | head -10
# logs directory should be writable by jPTS process
```

**Force rotation**:
```bash
# Restart application to trigger rotation check
./bin/q2 stop
./bin/q2  # Start again
```

#### **Issue: Cannot Read Log File**

```bash
# Check file locks (Linux)
lsof | grep jpts.log

# If file is locked by running process, use `tail` instead of `less`
tail -f jpts.log
```

#### **Issue: Out of Disk Space**

```bash
# Check available space
df -h

# Calculate total log size
du -sh logs/

# If needed, compress and archive immediately
for file in logs/jpts.log.{5..10}; do
  gzip "$file"
  mv "$file.gz" /backup/
done
```

### Log Analysis Tools

#### **Using grep for Pattern Matching**

```bash
# Case-insensitive error search
grep -i "error\|exception\|fail" jpts.log*

# Count occurrences per log file
grep -h "TXNNAME" jpts.log* | wc -l

# Show line numbers
grep -n "PIN translation" jpts.log
```

#### **Using awk for Advanced Analysis**

```bash
# Extract timestamp and message from each line
awk -F'[:]' '{print $1":"$2":"$3, $NF}' jpts.log | head -20

# Count messages by component
awk -F' - ' '{print $1}' jpts.log | sort | uniq -c | sort -rn
```

#### **Using sed for Filtering**

```bash
# Remove ANSI color codes
sed 's/\x1b\[[0-9;]*m//g' jpts.log

# Extract specific date range
sed -n '/2025-12-04 10:00:/,/2025-12-04 11:00:/p' jpts.log
```

### Log Backup and Recovery

#### **Create Backup Schedule**

**Cron job** (add to `/etc/crontab`):
```bash
# Daily backup at 2 AM
0 2 * * * cp /modules/jpts-app/build/install/jpts-app/logs/jpts.log /backup/jpts-$(date +\%Y\%m\%d).log
```

#### **Restore Logs**

```bash
# If logs are deleted accidentally
cp /backup/jpts-2025-12-04.log /modules/jpts-app/build/install/jpts-app/logs/jpts.log
```

---

## Summary: Monitoring and Logs

### Quick Reference

**Real-Time Monitoring**:
```bash
tail -f logs/jpts.log                          # Follow live logs
grep "ERROR" logs/jpts.log                     # Find errors
grep "id=12345" logs/jpts.log                  # Find transaction
grep "TXNNAME" logs/jpts.log                   # Find routing decisions
```

**Archive Retrieval**:
```bash
ls -lh logs/jpts.log*                          # List all logs
grep "id=12345" logs/jpts.log*                 # Search across archives
zcat logs/jpts.log.gz | grep ERROR             # Decompress and search
```

**Analysis**:
```bash
grep -c "ERROR" logs/jpts.log*                 # Count errors
grep "CreateTranLog" logs/jpts.log | wc -l     # Count transactions
awk -F' - ' '{print $1}' logs/jpts.log | sort | uniq -c  # Component summary
```

**Key Metrics to Monitor**:
- Transaction success rate
- Response time per destination
- Error frequency
- PIN translation failures
- Database connectivity
- Route selection accuracy

**Next Step**: Move to Task 4 (Error Codes) for understanding error conditions.
