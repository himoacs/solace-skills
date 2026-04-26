---
name: solace-semp-review
description: 'Solace SEMP API usage audit. Use for reviewing SEMP API security, automation scripts, API versioning, error handling, pagination patterns, and identifying SEMP usage anti-patterns.'
argument-hint: 'security, automation, or best-practices'
---

# Solace SEMP Review

Expert guidance for auditing SEMP API usage and automation scripts.

## When to Use

- **Automation Review**: Audit SEMP scripts and tools
- **Security Audit**: Review SEMP access patterns
- **API Best Practices**: Validate SEMP usage
- **Migration**: Update to SEMP v2

## SEMP Audit Checklist

### Security

```markdown
## SEMP Security Checklist

### Authentication
- [ ] No hardcoded credentials in scripts
- [ ] Using service accounts, not personal accounts
- [ ] Credentials stored in secrets manager
- [ ] HTTPS used for SEMP calls
- [ ] Basic auth or OAuth (not anonymous)

### Authorization
- [ ] Minimum required permissions
- [ ] Read-only accounts for monitoring
- [ ] Separate accounts per automation
- [ ] Admin access limited and audited

### Network
- [ ] SEMP port restricted to management network
- [ ] TLS enabled for SEMP
- [ ] IP allowlisting if possible
```

### API Usage

```markdown
## SEMP API Usage Checklist

### Version
- [ ] Using SEMP v2 (not legacy v1)
- [ ] Consistent API version across scripts
- [ ] Version pinned where needed

### Error Handling
- [ ] All responses checked for errors
- [ ] Specific error codes handled
- [ ] Retries with backoff implemented
- [ ] Rate limiting respected

### Efficiency
- [ ] Pagination used for large results
- [ ] Selective fields requested
- [ ] Caching where appropriate
- [ ] Batch operations used when possible
```

## SEMP Best Practices

### Secure Credentials

```python
# ✅ GOOD: Credentials from environment/secrets
import os
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

# Option 1: Environment variables
semp_user = os.environ['SOLACE_SEMP_USER']
semp_pass = os.environ['SOLACE_SEMP_PASS']

# Option 2: Secrets manager
credential = DefaultAzureCredential()
client = SecretClient(vault_url="https://myvault.vault.azure.net/", credential=credential)
semp_pass = client.get_secret("solace-semp-password").value

# ❌ BAD: Hardcoded credentials
semp_user = "admin"
semp_pass = "admin123"  # Never do this!
```

### Proper Error Handling

```python
import requests
from requests.exceptions import RequestException
import time

class SempClient:
    def __init__(self, host, username, password):
        self.base_url = f"https://{host}:943/SEMP/v2"
        self.auth = (username, password)
        self.session = requests.Session()
        self.session.verify = True  # Verify SSL
    
    def request(self, method, endpoint, **kwargs):
        url = f"{self.base_url}/{endpoint}"
        
        for attempt in range(3):  # Retry logic
            try:
                response = self.session.request(
                    method, url, auth=self.auth, **kwargs
                )
                
                # Check for SEMP-specific errors
                if response.status_code == 200:
                    return response.json()
                elif response.status_code == 400:
                    error = response.json().get('meta', {}).get('error', {})
                    raise SempValidationError(error.get('description'))
                elif response.status_code == 401:
                    raise SempAuthenticationError("Invalid credentials")
                elif response.status_code == 403:
                    raise SempAuthorizationError("Insufficient permissions")
                elif response.status_code == 404:
                    return None  # Resource not found
                elif response.status_code == 429:
                    # Rate limited - wait and retry
                    time.sleep(2 ** attempt)
                    continue
                elif response.status_code >= 500:
                    # Server error - retry
                    time.sleep(2 ** attempt)
                    continue
                else:
                    response.raise_for_status()
                    
            except RequestException as e:
                if attempt == 2:
                    raise SempConnectionError(f"Failed after 3 attempts: {e}")
                time.sleep(2 ** attempt)
        
        raise SempError("Max retries exceeded")
```

### Pagination

```python
def get_all_queues(self, vpn):
    """Get all queues with proper pagination."""
    queues = []
    params = {'count': 100}  # Request 100 at a time
    
    while True:
        response = self.request(
            'GET', 
            f"config/msgVpns/{vpn}/queues",
            params=params
        )
        
        if not response:
            break
            
        data = response.get('data', [])
        queues.extend(data)
        
        # Check for next page
        paging = response.get('meta', {}).get('paging', {})
        cursor = paging.get('cursorQuery')
        
        if cursor:
            params = {'cursor': cursor}
        else:
            break
    
    return queues
```

### Selective Fields

```python
# ✅ GOOD: Request only needed fields
response = client.request(
    'GET',
    'monitor/msgVpns/default/queues',
    params={'select': 'queueName,msgCount,bindCount,msgSpoolUsage'}
)

# ❌ BAD: Request everything when only need a few fields
response = client.request('GET', 'monitor/msgVpns/default/queues')
# Returns 50+ fields per queue
```

## SEMP Anti-Patterns

### 1. No Error Handling

```python
# ❌ BAD: No error handling
def create_queue(queue_name):
    response = requests.post(
        f"{SEMP_URL}/config/msgVpns/default/queues",
        auth=(user, password),
        json={"queueName": queue_name}
    )
    # Assumes success - will fail silently!
```

### 2. Polling Without Backoff

```python
# ❌ BAD: Aggressive polling
while True:
    stats = get_queue_stats()
    process(stats)
    # No sleep! Hammers the API

# ✅ GOOD: Respectful polling
while True:
    stats = get_queue_stats()
    process(stats)
    time.sleep(5)  # Wait between polls
```

### 3. Not Using Pagination

```python
# ❌ BAD: Assumes all data in one response
response = requests.get(f"{SEMP_URL}/config/msgVpns/default/queues")
queues = response.json()['data']  # May be incomplete!
```

### 4. Using SEMP v1

```python
# ❌ BAD: Legacy SEMP v1
response = requests.post(
    f"http://broker:8080/SEMP",
    data="<rpc semp-version='soltr/8_4_0'>...</rpc>"  # XML-based, deprecated
)

# ✅ GOOD: SEMP v2
response = requests.get(
    f"https://broker:943/SEMP/v2/config/msgVpns",
    auth=(user, password)
)
```

### 5. Hardcoded Versions

```python
# ❌ BAD: Hardcoded SEMP version that may not exist
url = "https://broker:943/SEMP/v2.35/config/..."  # May not match broker version

# ✅ GOOD: Use unversioned path or check supported versions
url = "https://broker:943/SEMP/v2/config/..."

# Or dynamically determine supported version
about = requests.get(f"https://broker:943/SEMP/v2/about/api").json()
version = about['data']['sempVersion']
```

## SEMP Security Audit

### Check SEMP Access

```bash
#!/bin/bash
# Audit SEMP configuration

echo "=== SEMP Security Audit ==="

# Check if TLS is enabled
echo -n "SEMP TLS: "
curl -s -o /dev/null -w "%{http_code}" "https://$BROKER:943/SEMP/v2/about/api" 2>/dev/null
if [ $? -eq 0 ]; then
    echo "Enabled"
else
    echo "WARNING: TLS may not be enabled"
fi

# Check if anonymous access is blocked
echo -n "Anonymous access: "
status=$(curl -s -o /dev/null -w "%{http_code}" "$SEMP_URL/about/api")
if [ "$status" == "401" ]; then
    echo "Blocked (Good)"
else
    echo "WARNING: Anonymous access may be allowed"
fi

# Check admin accounts
echo "=== Admin Accounts ==="
curl -s -u $SEMP_AUTH "$SEMP_URL/config/clientUsernames" | \
jq '.data[] | select(.clientUsername | test("admin|root|superuser")) | .clientUsername'
```

### Monitor SEMP Usage

```bash
# Track SEMP API calls via event logs
grep -i "semp" /var/solace/logs/command.log | \
awk '{print $1, $2, $5, $6}' | \
sort | uniq -c | sort -rn | head -20

# Look for suspicious patterns:
# - Many failed auth attempts
# - Configuration changes at odd hours
# - Bulk operations
```

## SEMP Script Review Checklist

```markdown
## Script Review Checklist

### Security
- [ ] No hardcoded credentials
- [ ] Uses HTTPS
- [ ] Service account with minimal permissions
- [ ] Secrets from secure source

### Error Handling
- [ ] HTTP errors checked
- [ ] SEMP errors parsed
- [ ] Retries implemented
- [ ] Timeouts configured

### Efficiency
- [ ] Pagination implemented
- [ ] Only needed fields requested
- [ ] Caching where appropriate
- [ ] Rate limiting respected

### Maintainability
- [ ] Logging for troubleshooting
- [ ] Configuration externalized
- [ ] Version compatibility considered
- [ ] Tests included
```

## SEMP Review Report Template

```markdown
# SEMP API Audit Report

**Scope**: [scripts/tools reviewed]
**Date**: [date]
**Reviewer**: [name]

## Summary
- Scripts Reviewed: [count]
- Compliant: [count]
- Issues Found: [count]

## Security Findings
| Issue | Severity | Location | Recommendation |
|-------|----------|----------|----------------|
| Hardcoded credentials | Critical | script.py:15 | Use env vars |
| No TLS | High | config.yaml | Enable HTTPS |

## Best Practice Violations
| Issue | Severity | Location |
|-------|----------|----------|
| No pagination | Medium | get_queues.py |
| No error handling | High | create_queue.py |

## Recommendations
1. [Priority 1: Security fixes]
2. [Priority 2: Error handling]
3. [Priority 3: Efficiency improvements]
```

## Reference Links

- **SEMP v2 Documentation**: https://docs.solace.com/SEMP/Using-SEMP.htm
- **SEMP API Reference**: https://docs.solace.com/SEMP/SEMP-API-Reference.htm
- **SEMP Best Practices**: https://docs.solace.com/SEMP/SEMP-Best-Practices.htm

## Further Reading

- [SEMP Authentication](./references/semp-auth.md)
- [Automation Patterns](./references/automation.md)
- [Migration from v1](./references/semp-migration.md)
