---
name: solace-security-review
description: 'Solace security configuration audit. Use for reviewing authentication setup, ACL profiles, TLS configuration, credential management, audit logging, and identifying security vulnerabilities.'
argument-hint: 'authentication, acl, tls, or credentials'
---

# Solace Security Review

Expert guidance for auditing Solace PubSub+ security configurations.

## When to Use

- **Security Audit**: Review broker security posture
- **Compliance**: Validate against security standards
- **Pre-Production**: Security readiness check
- **Vulnerability Assessment**: Identify security gaps

## Security Audit Checklist

### Authentication

```markdown
## Authentication Audit Checklist

### Credentials
- [ ] Default admin password changed
- [ ] Strong password policy enforced
- [ ] Service accounts have unique credentials
- [ ] No shared credentials
- [ ] Credentials stored in secrets manager (not config)

### Authentication Schemes
- [ ] Appropriate auth scheme per use case
- [ ] OAuth/OIDC for user-facing apps
- [ ] Client certificates for service-to-service
- [ ] Basic auth only with TLS
- [ ] LDAP integration for enterprise users

### Client Usernames
- [ ] All enabled usernames documented
- [ ] Unused usernames disabled/deleted
- [ ] Each service has dedicated username
- [ ] No wildcard username patterns
```

### Authorization (ACLs)

```markdown
## ACL Audit Checklist

### Profile Configuration
- [ ] Default action is "disallow" for publish/subscribe
- [ ] Exceptions are specific, not broad
- [ ] No `>` wildcard in exceptions
- [ ] ACL profiles documented per service
- [ ] Principle of least privilege applied

### Access Control
- [ ] Each service has dedicated ACL profile
- [ ] Cross-service access explicitly authorized
- [ ] Topic namespaces properly segmented
- [ ] Queue access restricted to owners
- [ ] Admin access limited and audited
```

### Encryption

```markdown
## Encryption Audit Checklist

### TLS Configuration
- [ ] TLS enabled for all client connections
- [ ] TLS 1.2+ only (1.0/1.1 disabled)
- [ ] Strong cipher suites configured
- [ ] Plain-text ports disabled in production
- [ ] Certificate valid and not expired

### Certificate Management
- [ ] Certificates from trusted CA (not self-signed in prod)
- [ ] Certificate rotation process defined
- [ ] Private keys securely stored
- [ ] Certificate expiry monitoring in place
```

## Automated Security Checks

### Check Default Credentials

```bash
#!/bin/bash
# Check for default/weak configurations

# Test default admin password
if curl -s -o /dev/null -w "%{http_code}" \
   -u admin:admin "$SEMP_URL/config" | grep -q "200"; then
    echo "CRITICAL: Default admin password in use!"
fi

# Check for plain-text ports enabled
curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default" | \
jq 'if .data.serviceMqttPlainTextEnabled then "WARNING: MQTT plain-text enabled" else empty end'

curl -s -u $SEMP_AUTH "$SEMP_URL/config" | \
jq 'if .data.serviceSmfPlainTextEnabled then "WARNING: SMF plain-text enabled" else empty end'
```

### Audit ACL Profiles

```bash
#!/bin/bash
# Find overly permissive ACL profiles

echo "=== Checking ACL Profiles ==="

# Check for default allow
curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/aclProfiles" | \
jq '.data[] | select(.publishTopicDefaultAction == "allow") | 
    {profile: .aclProfileName, issue: "Default ALLOW for publish"}'

curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/aclProfiles" | \
jq '.data[] | select(.subscribeTopicDefaultAction == "allow") | 
    {profile: .aclProfileName, issue: "Default ALLOW for subscribe"}'

# Check for broad wildcards in exceptions
for profile in $(curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/aclProfiles" | jq -r '.data[].aclProfileName'); do
    curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/aclProfiles/$profile/publishTopicExceptions" | \
    jq --arg p "$profile" '.data[] | select(.publishTopicException | test("^>$|^[^/]+/>$")) |
        {profile: $p, topic: .publishTopicException, issue: "Overly broad exception"}'
done
```

### Check TLS Configuration

```bash
#!/bin/bash
# Validate TLS configuration

echo "=== TLS Configuration Audit ==="

# Check if TLS is enabled
curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default" | \
jq '{
  smfTlsEnabled: .data.serviceTlsEnabled,
  mqttTlsEnabled: .data.serviceMqttTlsEnabled,
  restTlsEnabled: .data.serviceRestIncomingTlsEnabled
}'

# Check cipher suites
curl -s -u $SEMP_AUTH "$SEMP_URL/config" | \
jq '{cipherSuites: .data.tlsCipherSuiteList}'

# Test TLS connection
echo "Testing TLS..."
openssl s_client -connect broker:55443 -tls1_2 < /dev/null 2>&1 | \
grep -E "Protocol|Cipher"
```

## Security Scoring

### Scoring Matrix

| Category | Weight | Criteria |
|----------|--------|----------|
| Authentication | 25% | Schemes, password policy, no defaults |
| Authorization | 25% | ACL profiles, least privilege |
| Encryption | 25% | TLS config, certificates |
| Audit | 15% | Logging, monitoring |
| Network | 10% | Ports, firewall |

### Scoring Scale

| Score | Rating | Action |
|-------|--------|--------|
| 90-100 | Excellent | Minor improvements |
| 70-89 | Good | Address warnings |
| 50-69 | Fair | Remediation needed |
| <50 | Poor | Critical fixes required |

## Common Vulnerabilities

### 1. Default Credentials

```bash
# VULNERABILITY: Default admin password
# RISK: Critical
# CHECK:
curl -s -u admin:admin "$SEMP_URL/config" -o /dev/null -w "%{http_code}"
# If returns 200, default password is in use!
```

### 2. Plain-Text Ports

```bash
# VULNERABILITY: Unencrypted communication
# RISK: High
# CHECK:
curl -s -u $SEMP_AUTH "$SEMP_URL/config" | \
jq '{
  smfPlainText: .data.serviceSmfPlainTextEnabled,
  mqttPlainText: .data.serviceMqttPlainTextEnabled
}'
```

### 3. Overly Permissive ACLs

```bash
# VULNERABILITY: Default allow on ACL
# RISK: High
# CHECK:
curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default/aclProfiles/default" | \
jq '{
  publishDefault: .data.publishTopicDefaultAction,
  subscribeDefault: .data.subscribeTopicDefaultAction
}'
# Both should be "disallow"
```

### 4. Weak TLS Configuration

```bash
# VULNERABILITY: Weak ciphers or old TLS versions
# RISK: Medium
# CHECK:
# Test for TLS 1.0/1.1 support (should fail)
openssl s_client -connect broker:55443 -tls1 < /dev/null 2>&1
# Should return error "no protocols available"
```

### 5. Missing Audit Logging

```bash
# VULNERABILITY: No security event logging
# RISK: Medium
# CHECK:
curl -s -u $SEMP_AUTH "$SEMP_URL/config/msgVpns/default" | \
jq '{
  clientConnectEvents: .data.eventClientConnectEnabled,
  clientDisconnectEvents: .data.eventClientDisconnectEnabled
}'
# Both should be true
```

## Security Review Report Template

```markdown
# Solace Security Audit Report

**System**: [name]
**Date**: [date]
**Auditor**: [name]
**Classification**: [Confidential]

## Executive Summary
- Overall Security Score: [X/100]
- Critical Findings: [count]
- High-Risk Findings: [count]
- Compliance Status: [Pass/Fail]

## Authentication Audit
| Check | Status | Finding |
|-------|--------|---------|
| Default credentials | ✅/❌ | |
| Password policy | ✅/❌ | |
| Auth scheme | ✅/❌ | |

## Authorization Audit
| Check | Status | Finding |
|-------|--------|---------|
| ACL default deny | ✅/❌ | |
| Least privilege | ✅/❌ | |
| Topic segmentation | ✅/❌ | |

## Encryption Audit
| Check | Status | Finding |
|-------|--------|---------|
| TLS enabled | ✅/❌ | |
| Strong ciphers | ✅/❌ | |
| Valid certificates | ✅/❌ | |

## Findings

### Critical
| ID | Finding | Risk | Remediation |
|----|---------|------|-------------|
| C1 | [finding] | Critical | [action] |

### High
| ID | Finding | Risk | Remediation |
|----|---------|------|-------------|
| H1 | [finding] | High | [action] |

## Remediation Plan
| Priority | Finding | Owner | Due Date |
|----------|---------|-------|----------|
| 1 | [finding] | [owner] | [date] |

## Compliance Mapping
| Standard | Requirement | Status |
|----------|-------------|--------|
| SOC 2 | Access Control | ✅/❌ |
| PCI DSS | Encryption | ✅/❌ |
```

## Reference Links

- **Security Overview**: https://docs.solace.com/Security/Security-Overview.htm
- **ACL Configuration**: https://docs.solace.com/Security/ACL-Profiles.htm
- **TLS Setup**: https://docs.solace.com/Security/TLS-SSL-Service-Configuration.htm

## Further Reading

- [Hardening Guide](./references/hardening.md)
- [Compliance Checklist](./references/compliance.md)
- [Incident Response](./references/incident-response.md)
