# Task 5: PIN Translation
## Deliverables: Understanding of PIN Translation

---

## 5.1 Test Environment PIN Translation (1 Hr)

### Overview

PIN (Personal Identification Number) translation is a critical security feature that converts a PIN from one encryption format to another without exposing the plaintext PIN. This is necessary when transactions flow from a Source Station (using one security module) to a Destination Station (using a different security module).

### PIN Translation Architecture

#### **Why PIN Translation is Needed**

```
POS Terminal
    ↓
    └─→ Encrypts PIN with Source Key
        ↓
        └─→ ISO message sent with encrypted PIN (Field 52)
            ↓
            └─→ Source Station receives
                ↓
                └─→ Must re-encrypt for JPTS Core
                    ↓ (Different security module = Different key)
                    ↓
                    └─→ JPTS Core
                        ↓
                        └─→ Must re-encrypt again for Destination Station
                            ↓ (Another different key)
                            ↓
                            └─→ Destination (ACI/Abiy/etc.)
```

**Problem**: If we decrypt to plaintext at any step, PIN is exposed to attack
**Solution**: Use PIN Translation to change encryption without decryption

### PIN Block Format

#### **PIN Block Structure** (ISO 9564-1 Format-0, most common)

```
PIN Block (8 bytes):
Byte 0:   0x[F/V] + PIN_LENGTH (nibbles)
Bytes 1-8: Encrypted PIN digits + PAN suffix + Padding

Example:
  PIN: 1234
  PAN: 4111111111111111

PIN Block (hex):
  04          (PIN length 4, Format 0)
  1234FFFFFF  (PIN digits + padding)
  11111111    (Last 12 PAN digits last 4)
  
  Total: 04 1234 FFFF FFFF 1111 1111 (8 bytes)
```

#### **PIN Block Encryption**

PIN blocks are encrypted with DES/3DES (for older systems) or AES (for modern systems):

```
Plain PIN Block (8 bytes)
    ↓ (Encrypt with DES/3DES using key)
Encrypted PIN Block (8 bytes, Field 52 in ISO 8583)
    ↓ (Transmitted in message)
```

### PIN Translation Process in jPTS

#### **Component: TranslatePin Participant**

**File**: `/modules/gateway/src/main/java/org/jpos/gw/TranslatePin.java`

**Purpose**: Translate PIN from source format to destination format

**Configuration** (in XML):
```xml
<participant class="org.jpos.gw.TranslatePin">
  <property name="request" value="REQUEST" />
  <property name="source-key" value="source-pin-key" />
  <property name="destination-key" value="dest-pin-key" />
  <property name="keystore" value="ks" />
  <property name="security-module" value="ssm" />
  <property name="validate-keys" value="true" />
</participant>
```

#### **Translation Flow**

```java
// From TranslatePin.java doPrepare() method
ISOMsg request = ctx.get(requestKey);  // REQUEST message with F52

if (request.hasField(52)) {            // PIN Block present
    CardHolder ch = new CardHolder(request);
    SMAdapter sm = getSecurityModule();            // Security module
    SecureKeyStore keystore = getKeystore();       // Key storage
    
    // Step 1: Get encryption keys
    SecureDESKey keyFrom = keystore.getKey(sourceKey);
    SecureDESKey keyTo = keystore.getKey(destKey);
    
    // Step 2: Validate keys (optional)
    if (validateKeys) {
        assertValidKey(sm, keyFrom);
        assertValidKey(sm, keyTo);
    }
    
    // Step 3: Translate PIN
    byte[] translatedPin = sm.translatePin(
        request.getBytes(52),           // Original encrypted PIN
        keyFrom,                         // Source key
        keyTo,                           // Destination key
        // PAN and other params
    );
    
    // Step 4: Update message with translated PIN
    request.setBytes(52, translatedPin);
}
```

### Test Environment Setup

#### **Step 1: Configure Security Module Simulator**

**File**: `devel.properties`

```properties
# Use HSM simulator for development
security_module=hsm-sim
key_store=ks

# Alternative production settings would point to real HSM
# security_module=hsm-real
# key_store=/path/to/real/keystore
```

#### **Step 2: Initialize Keys in Keystore**

**File**: Configuration or keystore initialization

```java
// Initialize keystore with test keys
SecureKeyStore ks = getKeystore("ks");

// Create test keys
SecureDESKey sourceKey = ks.generateKey("source-pin-key", "DES3");
SecureDESKey destKey = ks.generateKey("dest-pin-key", "DES3");

// For testing, keys can be static
ks.setKey("source-pin-key", new SecureDESKey(...));
ks.setKey("dest-pin-key", new SecureDESKey(...));
```

#### **Step 3: Configure Participant in Transaction Chain**

**Location**: Transaction manager configuration (q2 XML)

```xml
<txn name="auth-flow" class="org.jpos.transaction.TransactionManager">
  <participant class="org.jpos.gw.Incoming">
    <property name="station-type" value="SOURCE" />
  </participant>
  
  <!-- PIN Translation Participant -->
  <participant class="org.jpos.gw.TranslatePin">
    <property name="source-key" value="ss-key" />
    <property name="destination-key" value="jpts-key" />
    <property name="security-module" value="ssm" />
    <property name="keystore" value="ks" />
  </participant>
  
  <participant class="org.jpos.jpts.CreateTranLog" />
  
  <!-- Second PIN Translation for destination -->
  <participant class="org.jpos.gw.TranslatePin">
    <property name="source-key" value="jpts-key" />
    <property name="destination-key" value="aci-key" />
    <property name="security-module" value="ssm" />
    <property name="keystore" value="ks" />
  </participant>
  
  <participant class="org.jpos.gw.Outgoing">
    <property name="station-type" value="DESTINATION" />
  </participant>
</txn>
```

### Test Scenario: PIN Translation

#### **Scenario: Authorization with PIN**

**Test Case**: Translate PIN from Source Station to JPTS Core format

#### **Step 1: POS Generates PIN Block**

```
PIN: 1234
PAN: 4111111111111111

PIN Block (plain):  04 1234 FFFF FFFF 1111 1111

Encrypted with Source Key (ss-key):
  Result: 8F A2 C1 3D 9E 2B 4A 67  (example hex)
```

#### **Step 2: Source Station Sends to JPTS**

**Message Fields**:
```
F2:   4111111111111111     (Card PAN)
F52:  8FA2C13D9E2B4A67     (Encrypted PIN with ss-key)
```

#### **Step 3: TranslatePin Participant Executes**

```
In: F52 = 8FA2C13D9E2B4A67 (encrypted with ss-key)

Operation:
  1. Retrieve ss-key from keystore
  2. Retrieve jpts-key from keystore
  3. Call: sm.translatePin(
       encryptedPin: 8FA2C13D9E2B4A67,
       keyFrom: ss-key,
       keyTo: jpts-key
     )
  
Out: F52 = 7C 4E B9 5F 2D 8A 1C 9E  (encrypted with jpts-key)
```

#### **Step 4: Updated Message in JPTS**

**Message Fields**:
```
F2:   4111111111111111     (Card PAN)
F52:  7C4EB95F2D8A1C9E     (Encrypted PIN with jpts-key)
```

### Test Verification

#### **Test 1: Key Validation**

```bash
# From jPTS console, verify keys are loaded
q2> keylist
Keys in keystore:
  ss-key: Valid 3DES key
  jpts-key: Valid 3DES key
  aci-key: Valid 3DES key
```

#### **Test 2: PIN Translation Execution**

```bash
# Monitor PIN translation in logs
tail -f logs/jpts.log | grep "TranslatePin"

# Expected output:
# [Thread-1] DEBUG org.jpos.gw.TranslatePin - PIN translation started
# [Thread-1] DEBUG org.jpos.gw.TranslatePin - Source key: ss-key
# [Thread-1] DEBUG org.jpos.gw.TranslatePin - Destination key: jpts-key
# [Thread-1] DEBUG org.jpos.gw.TranslatePin - PIN translated successfully
```

#### **Test 3: Message Validation**

```java
// After translation, verify PIN block changed
ISOMsg original = ...  // With F52 = 8FA2C13D9E2B4A67
ISOMsg translated = ... // After TranslatePin

assert original.getBytes(52) != translated.getBytes(52):
  "PIN should be re-encrypted"

assert translated.getBytes(52).length == 8:
  "PIN block should be 8 bytes"
```

#### **Test 4: End-to-End Transaction**

```bash
# Send test authorization with PIN
./bin/test-txn \
  --card=4111111111111111 \
  --pin=1234 \
  --amount=100 \
  --type=auth

# Expected flow:
# 1. POS sends message with PIN (encrypted with ss-key)
# 2. Source Station receives, prepares for JPTS
# 3. TranslatePin #1: ss-key → jpts-key
# 4. JPTS processes transaction
# 5. TranslatePin #2: jpts-key → aci-key
# 6. Sends to ACI with ACI-encrypted PIN
# 7. Response: 00 (Approved)
```

### Debugging PIN Translation Issues

#### **Issue 1: "Invalid PIN" Error**

**Log**:
```
[ERROR] PIN validation failed after translation
```

**Cause**: PIN changed incorrectly during translation

**Debug Steps**:
```bash
1. Verify source and destination keys are different
2. Check key validity
3. Verify PIN block format (8 bytes)
4. Test with known PIN values
```

**Resolution**:
```bash
# Restart and reload keys
q2> reload-keys
q2> keylist  # Verify keys loaded
```

#### **Issue 2: "Key Not Found" Error**

**Log**:
```
[ERROR] Cannot find key: jpts-key
```

**Cause**: Key not configured or loaded in keystore

**Debug Steps**:
```bash
# Check configuration
grep "jpts-key" config/*.xml

# Check keystore
q2> keylist | grep jpts-key
```

**Resolution**:
```bash
# Generate/load key
q2> genkey jpts-key DES3
q2> loadkey jpts-key /path/to/key/file
```

#### **Issue 3: "Timeout" During Translation**

**Log**:
```
[ERROR] PIN translation timeout
```

**Cause**: Security module slow or hanging

**Debug Steps**:
```bash
# Check security module status
q2> ssm-status

# Check for stuck threads
jps -l | grep java
jstack <pid>
```

**Resolution**:
```bash
# Restart security module
q2> shutdown-security-module
q2> start-security-module
```

### Test Report Template

```
PIN Translation Test Report
===========================

Test Date: 2025-12-04
Environment: Development

Test Case 1: Basic PIN Translation
  Status: ✓ PASS / ✗ FAIL
  Input PIN Block (ss-key): 8FA2C13D9E2B4A67
  Output PIN Block (jpts-key): 7C4EB95F2D8A1C9E
  Duration: 45ms
  
Test Case 2: End-to-End Authorization
  Status: ✓ PASS / ✗ FAIL
  Card: 4111111111111111
  Amount: $100.00
  Result Code: 00
  Duration: 234ms
  
Test Case 3: Multiple PIN Translations
  Status: ✓ PASS / ✗ FAIL
  Translation 1 (ss→jpts): ✓
  Translation 2 (jpts→aci): ✓
  Duration: 89ms
  
Test Case 4: Invalid Key Handling
  Status: ✓ PASS / ✗ FAIL
  Expected Error: Key Not Found
  Received Error: Key Not Found ✓
  
Overall Result: ✓ ALL TESTS PASSED
Notes: PIN translation working correctly in test environment
```

---

## 5.2 Production Environment PIN Translation (1 Hr)

### Production Differences from Test Environment

#### **Key Differences**

| Aspect | Test | Production |
|--------|------|------------|
| Security Module | HSM Simulator (ssm) | Real HSM (Hardware) |
| Key Storage | In-memory keystore | Secure cryptographic token |
| Key Access | Direct programmatic | PIN-protected hardware access |
| PIN Format | Variable/flexible | Strict compliance |
| Compliance | Development focus | PCI-DSS Level 1 |
| Audit Trail | Optional | Mandatory logging |
| Key Management | Manual/admin | Automated/secure |

### HSM (Hardware Security Module) Integration

#### **Understanding HSM**

HSM is a physical device that:
- Generates and stores cryptographic keys
- Performs encryption/decryption operations
- Never exposes keys outside the device
- Provides tamper protection
- Creates audit logs

**Common HSM Models**:
- Thales Luna HSM
- Gemalto SafeNet
- Yubico YubiHSM
- nShield

#### **HSM Connection Configuration**

**Production Configuration**:
```properties
# Point to real HSM instead of simulator
security_module=hsm-real
hsm_provider=thales          # e.g., Thales Luna
hsm_host=hsm.example.com
hsm_port=5696
hsm_pin=protected-pin-xxxx    # Encrypted in secure storage
hsm_slot=0                    # HSM slot number
key_store=production-ks       # Refers to HSM-backed keystore
```

#### **PIN Translation with Real HSM**

```java
// Production PIN Translation
public class ProductionTranslatePin extends TxnSupport {
    
    @Override
    protected int doPrepare(long id, Context ctx) throws Exception {
        ISOMsg request = ctx.get(requestKey);
        
        if (request.hasField(52)) {
            // Get real HSM security module
            SMAdapter sm = getRealHSMModule();  // Connects to physical HSM
            
            // Keys are never extracted from HSM
            // Only HSM handles encryption/decryption
            SecureDESKey sourceKey = keystore.getKey(sourceKeyName);
            SecureDESKey destKey = keystore.getKey(destKeyName);
            
            // PIN translation happens entirely within HSM
            byte[] translatedPin = sm.translatePin(
                request.getBytes(52),
                sourceKey,
                destKey
                // PIN block never exposed outside HSM
            );
            
            // Only the encrypted result is returned
            request.setBytes(52, translatedPin);
            
            // HSM logs this operation for audit
            auditLog.record("PIN_TRANSLATION", sourceKeyName, destKeyName);
        }
        
        return PREPARED;
    }
}
```

### Production PIN Translation Workflow

#### **Step 1: Key Management**

**Key Generation (One-time)**:
```
HSM Administrator:
1. Creates source-pin-key in HSM
2. Creates dest-pin-key in HSM
3. Exports key checksums for verification
4. Never exports plaintext keys
```

**Key Verification**:
```
System Administrator:
1. Receives key checksums
2. Compares with expected values
3. Confirms keys are correct
4. Keys never leave HSM
```

#### **Step 2: PIN Translation Request**

**Request Flow**:
```
Transaction with PIN (Field 52: 8FA2C13D9E2B4A67)
    ↓
    └─→ HSM Module receives request
        ↓
        └─→ Connect to physical HSM device
            ↓
            └─→ Authenticate with PIN
                ↓
                └─→ Send translation request
                    (only field 52 and key references)
                    ↓
                    └─→ HSM translates PIN internally
                        (no decryption to plaintext)
                        ↓
                        └─→ Returns encrypted PIN
                            (7C4EB95F2D8A1C9E)
```

#### **Step 3: Audit Logging**

**Mandatory Audit Trail** (for PCI-DSS compliance):

```sql
-- HSM Audit Table
INSERT INTO hsm_audit_log (
    timestamp,
    operation,
    source_key_id,
    dest_key_id,
    status,
    pan_last_4,
    result
) VALUES (
    NOW(),
    'PIN_TRANSLATION',
    'source-pin-key',
    'dest-pin-key',
    'SUCCESS',
    '1111',  -- Last 4 digits only
    'TRANSLATED'
);
```

### Production Security Considerations

#### **1. PIN Block Protection**

**Encryption Methods**:
```
Source Encryption: 3DES or AES
    ↓
    └─→ Encrypted PIN Block (Field 52)
        ↓
        └─→ Travels in ISO 8583 message
            ↓
            └─→ Never decrypted (only translated)
                ↓
                └─→ Final encryption: 3DES or AES
                    ↓
                    └─→ Delivered to acquirer
```

#### **2. Key Rotation**

**Regular Key Rotation Policy**:
```
Every 90 days (or as per compliance):
1. Generate new PIN encryption key
2. Create translation matrix (old_key → new_key)
3. Batch-translate all stored PINs
4. Update key references
5. Securely retire old key
6. Audit log entry
```

**Implementation**:
```java
public void rotatePINKey(String oldKeyId, String newKeyId) {
    // Get all transactions with PINs from past 90 days
    List<TranLog> transactions = txnDao.findWithPINs(
        LocalDateTime.now().minusDays(90),
        LocalDateTime.now()
    );
    
    // Translate each PIN
    for (TranLog txn : transactions) {
        byte[] originalPin = txn.getEncryptedPin();  // Encrypted with oldKey
        
        // Translate via HSM
        byte[] newPin = hsm.translatePin(
            originalPin,
            keystore.getKey(oldKeyId),
            keystore.getKey(newKeyId)
        );
        
        // Update database
        txn.setEncryptedPin(newPin);
        txn.setKeyVersion(newKeyId);
        txnDao.save(txn);
        
        // Log rotation
        auditLog.record("PIN_ROTATION", oldKeyId, newKeyId, txn.getId());
    }
}
```

#### **3. PCI-DSS Compliance**

**PIN Translation Compliance Checklist**:

```
✓ PIN never stored in plaintext
✓ PIN only exists in encrypted form
✓ PIN translation performed in HSM (never in software)
✓ All PIN operations logged and auditable
✓ Access to HSM restricted and monitored
✓ Keys protected with 2FA/multi-party control
✓ Regular key rotation implemented
✓ Incident response plan documented
✓ Regular security assessments conducted
✓ Staff training on PIN handling procedures
```

#### **4. Multi-Party Control (MPC)**

**MPC Implementation**:
```
Scenario: 3-person MPC for PIN key decryption

Step 1: Three designated officers
  - Officer A: Has key share 1
  - Officer B: Has key share 2
  - Officer C: Has key share 3

Step 2: HSM Key Escrow
  When PIN translation needed:
  - All three officers required
  - Each enters share at HSM console
  - HSM reconstructs key only in memory
  - Operation proceeds
  - Audit log records all three participants

Step 3: Audit Trail
  Entry: "PIN_OPERATION: Officers A,B,C; Timestamp: 2025-12-04 10:30:15 UTC"
```

### Production PIN Translation Monitoring

#### **Performance Metrics**

```sql
-- Monitor PIN translation performance
SELECT 
    DATE_TRUNC('hour', timestamp) as hour,
    COUNT(*) as translation_count,
    AVG(duration_ms) as avg_duration,
    MAX(duration_ms) as max_duration,
    COUNT(CASE WHEN status = 'ERROR' THEN 1 END) as error_count
FROM hsm_audit_log
WHERE operation = 'PIN_TRANSLATION'
    AND timestamp > NOW() - INTERVAL '24 hours'
GROUP BY hour
ORDER BY hour DESC;
```

**Expected Output**:
```
Hour            Count  Avg Duration  Max Duration  Errors
2025-12-04 10   1245   125ms        342ms         0
2025-12-04 09   1123   118ms        301ms         0
2025-12-04 08   998    122ms        289ms         1
```

#### **Alert Thresholds**

```
Critical Alerts:
  - PIN translation duration > 500ms
  - PIN translation error rate > 0.5%
  - HSM connection timeout
  - Unauthorized HSM access attempt
  - Key mismatch detected
  
Warning Alerts:
  - PIN translation duration > 300ms
  - Key approaching rotation date
  - Audit log near capacity
```

### Production Incident Handling

#### **Incident: HSM Connection Loss**

```
Timeline:
  10:30:15 - Last successful PIN translation
  10:30:45 - HSM connection timeout detected
  10:30:46 - Alert triggered
  
Response Plan:
  1. Log error: "HSM_CONNECTION_FAILED"
  2. Fail-over to backup HSM (if configured)
  3. Queue transactions for retry
  4. Notify operations team
  5. Attempt reconnection every 5 seconds
  6. After 5 min: Page on-call security team
  7. Manual fallback: Use backup key from secure storage
  
Recovery:
  1. Investigate connection cause
  2. Verify HSM operational
  3. Re-establish connection
  4. Process queued transactions
  5. Post-incident review
  6. Update monitoring thresholds if needed
```

#### **Incident: PIN Translation Failure**

```
Scenario: PIN translation returns ERROR for valid PIN

Cause Analysis:
  1. Check HSM logs for errors
  2. Verify key exists and is active
  3. Check PIN block format (8 bytes)
  4. Verify transaction hasn't been modified
  5. Check for concurrent modification

Resolution:
  1. If key issue: Rotate or reload key
  2. If format issue: Validate message format
  3. If HSM issue: Restart or failover
  4. Retry transaction
  5. If continues: Escalate to security team
```

### Production PIN Translation Testing

#### **Pre-Production Testing**

```
Test Environment → Staging → Production

Staging Tests (replicate production):
  1. Real HSM connection (test HSM)
  2. Production key format
  3. Production audit logging
  4. Production compliance checks
  5. Load testing (1000s PIN/sec)
  6. Failover testing
  7. Recovery procedures
```

#### **Production Validation**

```bash
# Weekly validation
./validate-pin-translation.sh

# Checks:
1. HSM connectivity - PASS
2. Key availability - PASS
3. PIN translation latency < 100ms - PASS
4. Audit logs recording - PASS
5. Error rate < 0.1% - PASS
6. Key checksums match - PASS
```

### Comparison: Test vs Production

#### **PIN Translation Comparison**

| Component | Test | Production |
|-----------|------|-----------|
| HSM | Simulator in software | Physical device |
| Key Storage | Program memory | Cryptographic token |
| Key Access | Direct | Authentication required |
| Translation Speed | Varies | Consistent |
| Audit Trail | Optional | Mandatory |
| Compliance | Development | PCI-DSS Level 1 |
| Failover | None | Automatic |
| Recovery | Manual | Automated |
| Monitoring | Basic logs | Advanced metrics |
| Incident Response | Debug logs | Escalation procedures |

---

## Summary: PIN Translation

### Key Concepts

1. **What**: PIN Translation converts encrypted PIN from one format to another
2. **Why**: Allows PIN to flow through multiple security domains
3. **How**: Use HSM to translate without exposing plaintext
4. **Where**: In TranslatePin participant in transaction flow
5. **When**: Every time transaction crosses security boundary

### PIN Translation Flow (Complete)

```
POS PIN Input (Plaintext)
    ↓ (User enters PIN)
    ├─→ POS encrypts with ss-key
        ↓
        ├─→ ISO Message F52 = encrypted PIN
            ↓
            ├─→ TranslatePin #1: ss-key → jpts-key
                ↓
                ├─→ JPTS Core processes
                    ↓
                    ├─→ TranslatePin #2: jpts-key → aci-key
                        ↓
                        ├─→ Sent to ACI
                            ↓
                            ├─→ ACI decrypts with aci-key
                                ↓
                                ├─→ PIN Verification
                                    ↓
                                    ├─→ Response: 00 (Approved) or 55 (Declined)
```

### Security Best Practices

**Test Environment**:
- Use HSM simulator for development
- Static keys for reproducibility
- Flexible testing scenarios
- Basic audit logging

**Production Environment**:
- Real HSM with tamper protection
- Regular key rotation (every 90 days)
- Multi-Party Control (3+ signatures)
- Comprehensive audit trail
- Strict access controls
- PCI-DSS compliance
- Failover HSM for redundancy
- 24/7 monitoring and alerting

### Troubleshooting Quick Reference

| Problem | Cause | Solution |
|---------|-------|----------|
| PIN translation fails | HSM unavailable | Check HSM connection |
| PIN validation fails | Wrong key used | Verify key assignment |
| Timeout during translation | HSM slow | Check HSM performance |
| "Key not found" error | Key not loaded | Load key from HSM |
| PIN block corrupted | Bad encryption | Validate PIN format |

### Documentation and Compliance

**Required Documentation**:
- PIN translation procedure manual
- HSM configuration guide
- Key management procedures
- Incident response plan
- Audit trail procedures
- Staff training materials
- PCI-DSS compliance evidence

**Regulatory References**:
- PCI-DSS Requirement 2.3 (PIN handling)
- PCI-DSS Requirement 3.2 (Key management)
- ISO 16609 (PIN verification algorithm)
- ISO 9564 (PIN format)

---

## Final Task Completion

All five tasks completed:

✓ **Task 1: System Setup** - Architecture, modules, database, configuration
✓ **Task 2: Transaction Routing** - Complete transaction flow from POS to acquirer
✓ **Task 3: Monitoring and Logs** - Real-time and archived log monitoring
✓ **Task 4: Error Codes** - ISO 8583 and protocol-specific error handling
✓ **Task 5: PIN Translation** - Test and production PIN security

### Next Steps

1. **Review all documentation** for completeness
2. **Set up local development environment** using Task 1
3. **Execute sample transactions** following Task 2
4. **Monitor system** using Task 3 techniques
5. **Handle errors** based on Task 4 reference
6. **Implement PIN translation** per Task 5 guidelines
7. **Conduct team training** on complete system

### Contact and Support

For questions on any task:
1. Check relevant task documentation
2. Review source code comments
3. Check system logs (Task 3)
4. Consult error code reference (Task 4)
5. Escalate to security team for PIN issues (Task 5)
