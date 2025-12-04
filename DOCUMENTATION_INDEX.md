# flocash-jPTS Complete Documentation Index
## All Tasks Completed - Quick Reference Guide

---

## 📋 Documentation Overview

This directory contains comprehensive documentation for the flocash-jPTS payment transaction processing system. All five major tasks have been completed with detailed information on setup, routing, monitoring, error handling, and security.

### Documents Created

| Document | Task | Duration | Status |
|----------|------|----------|--------|
| TASK_1_SYSTEM_SETUP.md | System Setup | 7 Hrs | ✓ Complete |
| TASK_2_TRANSACTION_ROUTING.md | Transaction Routing | 5.5 Hrs | ✓ Complete |
| TASK_3_MONITORING_LOGS.md | Monitoring & Logs | 2 Hrs | ✓ Complete |
| TASK_4_ERROR_CODES.md | Error Codes | 1 Hr | ✓ Complete |
| TASK_5_PIN_TRANSLATION.md | PIN Translation | 2 Hrs | ✓ Complete |
| **TOTAL** | **5 Tasks** | **17.5 Hrs** | ✓ **Complete** |

---

## 🚀 Quick Start Guide

### For New Team Members

**Start Here**:
1. Read TASK_1 (System Setup) - Understand architecture
2. Run through Task 1.4 (System Setup) - Get system running
3. Read TASK_2 (Transaction Routing) - Understand flow
4. Read TASK_3 (Monitoring) - Learn to observe
5. Reference TASK_4 and TASK_5 as needed

**Expected Time**: 4-6 hours for full understanding

### For Operations/Support Team

**Focus On**:
- Task 3: Monitoring and Logs - Monitor live system
- Task 4: Error Codes - Handle issues
- Task 3.2: Archived Logs - Investigate historical issues

### For Developers

**Focus On**:
- Task 1: System Setup - Understand modules
- Task 2: Transaction Routing - Understand code flow
- Task 5: PIN Translation - Security implementation

### For Operations Engineers

**Focus On**:
- Task 1.4: System Setup - Deployment
- Task 3: Monitoring and Logs - Observability
- Task 4: Error Codes - Troubleshooting

---

## 📖 Document Summary

### Task 1: System Setup (TASK_1_SYSTEM_SETUP.md)

**What You'll Learn**:
- Complete system architecture
- Module purposes and relationships
- Database design and setup
- Build and deployment process
- Configuration management

**Key Sections**:
- 1.1: System Structure (1 Hr)
- 1.2: Modules Overview (3 Hrs)
- 1.3: Database Setup (1 Hr)
- 1.4: System Setup (2 Hrs)

**When You Need It**: Understanding system, setting up dev environment

**Key Files**: `/modules/*`, `devel.properties`, `build.gradle`

---

### Task 2: Transaction Routing (TASK_2_TRANSACTION_ROUTING.md)

**What You'll Learn**:
- How transactions flow through system
- POS to Source Station routing
- Source Station to JPTS Core routing
- JPTS Core to Destination Station routing
- Destination Station to Acquirer routing
- Simulator usage

**Key Sections**:
- 2.1: POS to Source Station (1 Hr)
- 2.2: Source Station to JPTS Core (30 Mins)
- 2.3: JPTS Core to Destination (1 Hr)
- 2.4: Destination to Acquirer (2 Hrs)
- 2.5: Using Simulator (1 Hr)

**When You Need It**: Debugging transactions, understanding flow

**Key Classes**: 
- `TransactionBuilder.java` - Source station enrichment
- `Incoming.java` - Message conversion to CMF
- `TxnName.java` - Routing decision
- `CreateTranLog.java` - Database logging
- `Outgoing.java` - Message conversion from CMF

---

### Task 3: Monitoring and Logs (TASK_3_MONITORING_LOGS.md)

**What You'll Learn**:
- Real-time log monitoring techniques
- Log file locations and rotation
- Archived log retrieval and analysis
- Log analysis patterns
- Performance monitoring

**Key Sections**:
- 3.1: Real-Time Log Monitoring (1 Hr)
- 3.2: Archived Logs Retrieval (1 Hr)

**When You Need It**: 
- Monitoring live system
- Debugging issues
- Analyzing historical data
- Performance tuning

**Key Commands**:
```bash
tail -f logs/jpts.log
grep "ERROR" logs/jpts.log
grep "id=12345" logs/jpts.log*
```

---

### Task 4: Error Codes (TASK_4_ERROR_CODES.md)

**What You'll Learn**:
- ISO 8583 standard error codes (00-99)
- CMF internal error codes
- Protocol-specific codes (ACI, Abiy, Ethswitch)
- Finding and interpreting error codes
- Troubleshooting guide

**Key Sections**:
- 4.1: Finding and Interpreting Error Codes (1 Hr)

**Error Code Categories**:
- 00-09: Success codes
- 10-19: Decline codes
- 20-99: System/network errors

**When You Need It**:
- Transaction declined
- System error occurred
- Understanding error message
- Troubleshooting issues

**Quick Reference**:
| Code | Meaning | Action |
|------|---------|--------|
| 00 | Approved | Process normally |
| 05 | Declined | Inform customer |
| 91 | Timeout | Retry transaction |

---

### Task 5: PIN Translation (TASK_5_PIN_TRANSLATION.md)

**What You'll Learn**:
- PIN translation concepts
- Why PIN translation needed
- PIN block format
- Test environment PIN translation
- Production environment PIN translation
- HSM integration
- Security best practices

**Key Sections**:
- 5.1: Test Environment PIN Translation (1 Hr)
- 5.2: Production Environment PIN Translation (1 Hr)

**When You Need It**:
- Implementing security
- Understanding PIN flow
- Setting up HSM
- Debugging PIN issues

**Key Class**: `TranslatePin.java`

---

## 🔍 Finding Information

### By Topic

#### Architecture & Design
→ Task 1: System Setup, Section 1.1

#### Modules & Components
→ Task 1: System Setup, Section 1.2

#### Database
→ Task 1: System Setup, Section 1.3

#### Building & Deploying
→ Task 1: System Setup, Section 1.4

#### Transaction Flow
→ Task 2: Transaction Routing, all sections

#### Monitoring & Observability
→ Task 3: Monitoring and Logs

#### Error Handling
→ Task 4: Error Codes

#### Security & PIN
→ Task 5: PIN Translation

### By Activity

**I need to...**

- **Set up the system**: Task 1, Section 1.4
- **Build the code**: Task 1, Section 1.4
- **Create database**: Task 1, Section 1.3
- **Send a test transaction**: Task 2, Section 2.5
- **Monitor live logs**: Task 3, Section 3.1
- **Find old logs**: Task 3, Section 3.2
- **Understand error**: Task 4, Section 4.1
- **Handle a timeout (91)**: Task 4 + Task 3
- **Implement PIN security**: Task 5, all sections
- **Deploy to production**: Task 1 + Task 5
- **Debug routing issue**: Task 2 + Task 3
- **Check transaction status**: Task 3 + Task 4

---

## 📊 System Architecture Quick View

```
┌─────────────────────────────────────────────────────────────┐
│                    ENTRY POINTS                              │
├──────────────────┬──────────────────┬──────────────────────┤
│  Source Station  │   REST API       │   Web App            │
│  (ss-wiseasy)    │   (restapi)      │   (app)              │
└─────────┬────────┴────────┬─────────┴──────────────┬────────┘
          │                 │                        │
          └─────────────────┼────────────────────────┘
                            ↓
                   ┌────────────────────┐
                   │   JPTS Core        │
                   │ ┌────────────────┐ │
                   │ │  Incoming      │ │ (Convert → CMF)
                   │ │  TxnName       │ │ (Derive name)
                   │ │  TranslatePin  │ │ (Translate PIN)
                   │ │  CreateTranLog │ │ (Log to DB)
                   │ └────────────────┘ │
                   └────────┬───────────┘
                            ↓
          ┌─────────────────┼──────────────────┐
          ↓                 ↓                  ↓
    ┌──────────┐     ┌──────────┐     ┌──────────┐
    │ DS-ACI   │     │ DS-Abiy  │     │ DS-Jcard │
    │ (ACI)    │     │ (Abiy)   │     │ (Local)  │
    └────┬─────┘     └────┬─────┘     └────┬─────┘
         │                │                │
         └────────────────┼────────────────┘
                          ↓
              ┌─────────────────────────┐
              │  External Acquirers     │
              │  (ACI/Abiy/Ethswitch)   │
              └─────────────────────────┘
```

---

## 💾 Database Schema Quick Reference

### Main Tables

**tranlog** - Transaction Logging
```sql
id, created, amount, currency, card_pan, processing_code,
source_station, destination, status, response_code,
result_msg, to_account, invoice_number, ds_transport_data
```

**cardproducts** - Card Product Definitions
```sql
id, name, active, pos, atm, moto, ecommerce, ds
```

**binrange** - Card BIN Ranges
```sql
id, start, end, weight, cardproduct
```

---

## 🔐 Security Architecture

```
PIN Encryption Flow:
  User Input (Plaintext)
      ↓
      ├→ Encrypt with Source Key (POS)
         ↓
         ├→ ISO Message F52 = Encrypted
            ↓
            ├→ TranslatePin #1 (ss-key → jpts-key)
               ↓
               ├→ JPTS Processing
                  ↓
                  ├→ TranslatePin #2 (jpts-key → acquirer-key)
                     ↓
                     ├→ Destination Station
                        ↓
                        └→ Never Decrypted (Only Translated)
```

**Key Points**:
- PIN never stored as plaintext
- Translation never exposes plaintext
- HSM handles all PIN operations (production)
- All PIN operations audited
- PCI-DSS Compliant

---

## 📈 Performance Considerations

### Typical Latencies

| Operation | Latency | Threshold |
|-----------|---------|-----------|
| Local participant | 10-50ms | 100ms |
| PIN translation | 20-100ms | 200ms |
| Send to acquirer | 100-500ms | 1000ms |
| Total transaction | 200-800ms | 5000ms |

### Bottlenecks & Solutions

| Bottleneck | Cause | Solution |
|-----------|-------|----------|
| Slow acquirer response | Network/processor load | Implement queuing |
| Database slow | Table lock or missing index | Analyze query plans |
| PIN translation slow | HSM overload | Upgrade HSM |

---

## 🛠️ Troubleshooting Quick Guide

### Cannot Start System
1. Check Java is installed: `java -version`
2. Check PostgreSQL running: `psql -V`
3. Check ports available: `netstat -an | grep 6022`
4. Check logs: `tail -f logs/jpts.log`

### Transactions Failing
1. Check error code (Task 4)
2. Check transaction log: `grep "id=TXNID" logs/jpts.log*`
3. Check database: `SELECT * FROM tranlog WHERE id=...`
4. Check destination status

### High Error Rate
1. Check acquirer connectivity
2. Check PIN translation (if errors are "5x" codes)
3. Review error code distribution
4. Check for configuration issues

### Performance Issues
1. Check transaction latency distribution
2. Check database performance
3. Check HSM (if PIN translation slow)
4. Check network connectivity

---

## 📝 Configuration Quick Reference

### devel.properties Key Settings

```properties
# Database
dbname=flocash_jpts
dbhost=localhost
dbport=5432
dbuser=jpos
dbpass=password

# Services
jpts_cmf_port=9999
jpts_cmf_timeout=30000
wiseasy_server_port=6022

# Security
security_module=hsm-sim (test) / hsm-real (prod)
key_store=ks

# Destinations
abiy_server_host=localhost
abiy_server_port=9997
```

---

## 🎓 Learning Paths

### Path 1: Developer (3 days)
- Day 1: Task 1 (System Setup) - 4 hours
- Day 2: Task 2 (Transaction Routing) - 5 hours
- Day 3: Task 5 (PIN Translation) - 2 hours

### Path 2: Operations (2 days)
- Day 1: Task 1 (System Setup 1.4) - 2 hours
- Day 1: Task 3 (Monitoring) - 2 hours
- Day 2: Task 4 (Error Codes) - 1 hour
- Day 2: Task 3.2 (Archived Logs) - 1 hour

### Path 3: Security (1 day)
- Morning: Task 5 (PIN Translation) - 2 hours
- Afternoon: Task 1.3 (Database), Task 4 - 2 hours

### Path 4: Full System (1 week)
- All tasks sequentially
- Hands-on lab for each task
- End-to-end transaction testing

---

## 🔗 Cross-References

### Task Dependencies

```
Task 1 (System Setup)
    ↓
Task 2 (Transaction Routing) + Task 3 (Monitoring)
    ↓
Task 4 (Error Codes)
    ↓
Task 5 (PIN Translation)
```

### Related Documentation

- ISO 8583 Standard - Financial Transaction Message Format
- PCI-DSS - Payment Card Industry Data Security Standard
- jPOS Documentation - Java Payment Operating System
- HSM Manuals - Hardware Security Module specifics

---

## 📞 Support Resources

### Internal Team

- **Architecture**: See Task 1, Section 1.1
- **Modules**: See Task 1, Section 1.2
- **Database**: See Task 1, Section 1.3
- **Routing**: See Task 2
- **Monitoring**: See Task 3
- **Errors**: See Task 4
- **Security**: See Task 5

### External Resources

- jPOS Documentation: https://jpos.org
- ISO 8583 Specifications
- PCI-DSS Requirements
- Payment Processing Standards

### Escalation Path

1. Check documentation (this file)
2. Check relevant Task document
3. Check logs (Task 3)
4. Check error codes (Task 4)
5. Escalate to team lead
6. For security issues: Contact security team

---

## ✅ Verification Checklist

### System Setup Verification

- [ ] Java 1.8+ installed
- [ ] Gradle installed
- [ ] PostgreSQL running
- [ ] Database created (flocash_jpts)
- [ ] User 'jpos' created with permissions
- [ ] Build successful: `gradle -Ptarget=devel clean installApp`
- [ ] Schema created
- [ ] Default card product inserted
- [ ] System starts without errors

### System Ready for Development

- [ ] All tests passing
- [ ] Sample transaction processed successfully
- [ ] Real-time logs visible
- [ ] Archived logs accessible
- [ ] Error codes documented
- [ ] PIN translation working
- [ ] All team trained on documentation

---

## 📄 Document Maintenance

**Last Updated**: December 4, 2025
**Version**: 1.0
**Status**: Complete

### Future Updates Needed

- [ ] Add performance optimization guide
- [ ] Add disaster recovery procedures
- [ ] Add more sample transactions
- [ ] Add monitoring dashboards guide
- [ ] Update with production experience

### Feedback & Improvements

Please report:
1. Typos or unclear sections
2. Missing information
3. Outdated procedures
4. New use cases

---

## 📚 Quick Command Reference

```bash
# Build and Deploy
gradle -Ptarget=devel clean installApp

# Start System
cd modules/jpts-app/build/install/jpts-app
./bin/q2

# Monitor Logs
tail -f logs/jpts.log
grep "ERROR" logs/jpts.log
grep "id=12345" logs/jpts.log*

# Check Database
psql -U jpos -d flocash_jpts
SELECT * FROM tranlog ORDER BY created DESC LIMIT 10;

# Stop System
./bin/q2 -c "shutdown"
```

---

## 🎯 Goals Achieved

✓ Task 1: System Setup - Understand architecture, set up locally
✓ Task 2: Transaction Routing - Understand complete transaction flow
✓ Task 3: Monitoring and Logs - Monitor and diagnose issues
✓ Task 4: Error Codes - Handle and troubleshoot errors
✓ Task 5: PIN Translation - Implement security correctly

### Next Milestones

1. Deploy to staging environment
2. Execute full transaction test suite
3. Set up production monitoring
4. Complete security audit
5. Train operations team
6. Go live monitoring

---

**For questions about specific topics, refer to the relevant task document listed above.**
