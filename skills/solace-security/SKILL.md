---
name: solace-security
description: 'Solace security configuration and hardening guidance. Use for TLS/SSL setup, authentication schemes (OAuth, client certificates, LDAP), ACL profiles, authorization groups, DMR security, and security audit configurations.'
argument-hint: 'tls, oauth, certificates, acl, ldap, or audit'
---

# Solace Security

Expert guidance for securing Solace PubSub+ deployments.

## When to Use

- **TLS Configuration**: Encrypt connections
- **Authentication**: OAuth, certificates, LDAP, RADIUS
- **Authorization**: ACLs, client profiles
- **Hardening**: Security best practices
- **Audit**: Logging and compliance

## TLS/SSL Configuration

### Server Certificate Setup

```bash
# Generate self-signed certificate (development only)
openssl req -x509 -newkey rsa:4096 \
  -keyout server.key -out server.crt \
  -days 365 -nodes \
  -subj "/CN=solace-broker.example.com"

# Convert to PKCS12 for Solace
openssl pkcs12 -export \
  -in server.crt -inkey server.key \
  -out server.p12 -name solace

# Upload via CLI
home
enable
configure
ssl
  server-certificate server.p12
  server-certificate-password <password>
  end
```

### Configure TLS Listeners (SEMP)

```bash
# Enable TLS for SMF
curl -X PATCH "$SEMP_URL/config/msgVpns/production" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "tlsAllowDowngradeToPlainTextEnabled": false
  }'

# Configure TLS service ports on broker
curl -X PATCH "$SEMP_URL/config" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "serviceSmfTlsListenPort": 55443,
    "serviceMsgBackboneTlsEnabled": true
  }'
```

### Client TLS Configuration (Java)

```java
SessionProperties sessionProps = new SessionProperties();
sessionProps.setProperty(SessionProperties.HOST, 
    "tcps://broker.example.com:55443");
sessionProps.setProperty(SessionProperties.VPN_NAME, "production");

// Trust store
sessionProps.setProperty(SessionProperties.SSL_TRUST_STORE, 
    "/path/to/truststore.jks");
sessionProps.setProperty(SessionProperties.SSL_TRUST_STORE_PASSWORD, 
    "changeit");

// Validate certificate
sessionProps.setProperty(SessionProperties.SSL_VALIDATE_CERTIFICATE, true);
sessionProps.setProperty(SessionProperties.SSL_VALIDATE_CERTIFICATE_HOST, true);
```

## Client Certificate Authentication

### Configure Broker

```bash
# Enable client certificate authentication
curl -X PATCH "$SEMP_URL/config/msgVpns/production" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "authenticationClientCertEnabled": true,
    "authenticationClientCertValidateDateEnabled": true
  }'

# Upload CA certificate
# Via CLI:
configure
ssl
  trusted-common-name MyRootCA certificate-authority
  client-certificate-authority myCA.pem
  end
```

### Client Configuration

```java
// Client certificate authentication
sessionProps.setProperty(SessionProperties.AUTHENTICATION_SCHEME, 
    SessionProperties.AUTHENTICATION_SCHEME_CLIENT_CERTIFICATE);

sessionProps.setProperty(SessionProperties.SSL_KEY_STORE, 
    "/path/to/client-keystore.jks");
sessionProps.setProperty(SessionProperties.SSL_KEY_STORE_PASSWORD, 
    "keystore-password");
sessionProps.setProperty(SessionProperties.SSL_PRIVATE_KEY_PASSWORD, 
    "key-password");
```

## OAuth 2.0 Authentication

### Configure OAuth Profile

```bash
# Create OAuth profile
curl -X POST "$SEMP_URL/config/msgVpns/production/authenticationOauthProfiles" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "oauthProfileName": "okta-oauth",
    "enabled": true,
    "clientId": "solace-broker",
    "clientSecret": "client-secret",
    "issuer": "https://dev-123456.okta.com/oauth2/default",
    "jwksUri": "https://dev-123456.okta.com/oauth2/default/v1/keys",
    "usernameClaimName": "sub"
  }'

# Enable OAuth on VPN
curl -X PATCH "$SEMP_URL/config/msgVpns/production" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "authenticationOauthEnabled": true,
    "authenticationOauthDefaultProviderName": "okta-oauth"
  }'
```

### Client OAuth Configuration

```java
// OAuth authentication
sessionProps.setProperty(SessionProperties.AUTHENTICATION_SCHEME, 
    SessionProperties.AUTHENTICATION_SCHEME_OAUTH2);
sessionProps.setProperty(SessionProperties.OAUTH2_ACCESS_TOKEN, 
    accessToken);
```

## LDAP Authentication

### Configure LDAP Profile

```bash
# Create LDAP profile
curl -X POST "$SEMP_URL/config/msgVpns/production/authorizationGroups" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "authorizationGroupName": "#ldap-group",
    "enabled": true,
    "aclProfileName": "default",
    "clientProfileName": "default"
  }'

# Configure LDAP server (broker-level)
# Via CLI:
configure
ldap
  ldap-profile myLdap
    search
      base-dn "dc=example,dc=com"
      filter "(sAMAccountName=$CLIENT_USERNAME)"
    admin
      admin-dn "cn=admin,dc=example,dc=com"
      admin-password <password>
    server
      server ldap.example.com
        port 636
        tls enabled
    end
```

## ACL Profiles

### Create ACL Profile

```bash
# Create restrictive ACL profile
curl -X POST "$SEMP_URL/config/msgVpns/production/aclProfiles" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "aclProfileName": "order-service-acl",
    "clientConnectDefaultAction": "allow",
    "publishTopicDefaultAction": "disallow",
    "subscribeTopicDefaultAction": "disallow"
  }'

# Add publish exception (allow)
curl -X POST "$SEMP_URL/config/msgVpns/production/aclProfiles/order-service-acl/publishTopicExceptions" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "publishTopicException": "acme/orders/>",
    "publishTopicExceptionSyntax": "smf"
  }'

# Add subscribe exception
curl -X POST "$SEMP_URL/config/msgVpns/production/aclProfiles/order-service-acl/subscribeTopicExceptions" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "subscribeTopicException": "acme/orders/>",
    "subscribeTopicExceptionSyntax": "smf"
  }'
```

### ACL Pattern Examples

```
# Allow specific domain
acme/orders/>

# Allow single level wildcard
acme/*/orders/created

# Allow specific client's response topic
responses/client-12345/>

# Multi-tenant isolation
tenant/customer-abc/>
```

## Client Profiles

### Resource Limits

```bash
# Create client profile with limits
curl -X POST "$SEMP_URL/config/msgVpns/production/clientProfiles" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "clientProfileName": "limited-client",
    "allowGuaranteedMsgSendEnabled": true,
    "allowGuaranteedMsgReceiveEnabled": true,
    "maxConnectionCountPerClientUsername": 10,
    "maxEgressFlowCount": 100,
    "maxIngressFlowCount": 100,
    "maxSubscriptionCount": 500,
    "maxTransactedSessionCount": 0,
    "maxTransactionCount": 0
  }'
```

### Assign Profile to User

```bash
curl -X PATCH "$SEMP_URL/config/msgVpns/production/clientUsernames/app-user" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "clientProfileName": "limited-client",
    "aclProfileName": "order-service-acl"
  }'
```

## Security Hardening

### Broker-Level Security

```bash
# Disable unnecessary services
curl -X PATCH "$SEMP_URL/config" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "serviceSmfPlainTextEnabled": false,
    "serviceWebPlainTextEnabled": false
  }'

# Change default admin password
curl -X PATCH "$SEMP_URL/config/clientUsernames/admin" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "password": "strong-new-password-here"
  }'
```

### Hardening Checklist

```markdown
## Solace Security Hardening Checklist

### Authentication
- [ ] Change default admin credentials
- [ ] Enable TLS on all listeners
- [ ] Disable plain-text ports
- [ ] Configure authentication (OAuth/LDAP/certs)
- [ ] Set password policies

### Authorization
- [ ] Create specific ACL profiles per service
- [ ] Use deny-by-default ACLs
- [ ] Create restrictive client profiles
- [ ] Limit connections per username

### Network
- [ ] Use TLS for all client connections
- [ ] Enable TLS for SEMP management
- [ ] Configure trusted CA certificates
- [ ] Restrict SEMP access by IP/network

### Audit
- [ ] Enable authentication event logging
- [ ] Configure syslog forwarding
- [ ] Set up monitoring alerts
- [ ] Regular security reviews
```

## Audit Logging

### Enable Event Logging

```bash
# Configure event thresholds
curl -X PATCH "$SEMP_URL/config/msgVpns/production" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "eventConnectionCountThreshold": {
      "clearPercent": 60,
      "setPercent": 80
    },
    "eventMsgSpoolUsageThreshold": {
      "clearPercent": 60,
      "setPercent": 80
    }
  }'
```

### Syslog Configuration (CLI)

```
configure
logging
  event
    all
      set-value info
    client
      set-value info
    authentication
      set-value info
  syslog
    host 10.0.0.100
      port 514
      protocol udp
    end
```

### Security Event Monitoring

```bash
# Monitor authentication events
curl -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/production/clients" \
  | jq '.data[] | select(.clientUsername != null) | {
    user: .clientUsername,
    address: .clientAddress,
    connected: .uptime
  }'

# Monitor ACL denials
curl -u $SEMP_AUTH "$SEMP_URL/monitor/msgVpns/production/clients" \
  | jq '.data[] | {
    user: .clientUsername,
    aclDeniedPublish: .dataRxMsgCount,
    aclDeniedSubscribe: .controlRxMsgCount
  }'
```

## DMR Security

### Inter-Broker TLS

```bash
# Configure DMR cluster with TLS
curl -X POST "$SEMP_URL/config/dmrClusters" \
  -u $SEMP_AUTH \
  -H "Content-Type: application/json" \
  -d '{
    "dmrClusterName": "prod-cluster",
    "enabled": true,
    "tlsServerCertEnforceTrustedCommonNameEnabled": true,
    "authenticationBasicEnabled": true,
    "authenticationBasicPassword": "cluster-secret"
  }'
```

## Reference Links

- **Security Features**: https://docs.solace.com/Security/Security-Overview.htm
- **TLS Configuration**: https://docs.solace.com/Security/TLS-SSL-Service-Configuration.htm
- **ACL Profiles**: https://docs.solace.com/Security/ACL-Profiles.htm
- **OAuth Authentication**: https://docs.solace.com/Security/OAuth-Authentication.htm

## Further Reading

- [Certificate Management](./references/certificates.md)
- [OAuth Setup](./references/oauth-setup.md)
- [Compliance Checklist](./references/compliance.md)
---
name: solace-security
description: 'Solace security configuration guidance. Use for TLS/SSL setup, client authentication (basic, certificate, OAuth, LDAP), ACL profiles, authorization patterns, encryption at rest, and security audit logging.'
argument-hint: 'tls, oauth, ldap, acl, certificates, or audit'
---

# Solace Security

Expert guidance for securing Solace PubSub+ deployments.

## When to Use

- **Authentication Setup**: Client identity verification
- **Authorization**: ACL profiles and permissions
- **Encryption**: TLS/SSL configuration
- **Compliance**: Audit logging and security policies
- **OAuth Integration**: Token-based authentication

## Authentication Methods

### Basic Authentication

```bash
# Create client username
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/clientUsernames" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "clientUsername": "order-service",
    "enabled": true,
    "password": "secure-password-here",
    "aclProfileName": "order-service-acl",
    "clientProfileName": "default"
  }'
```

### Client Certificate Authentication

```bash
# 1. Enable certificate authentication on VPN
curl -X PATCH "http://broker:8080/SEMP/v2/config/msgVpns/default" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "authenticationClientCertEnabled": true,
    "authenticationClientCertAllowApiProvidedUsernameEnabled": false
  }'

# 2. Upload CA certificate
curl -X POST "http://broker:8080/SEMP/v2/config/certAuthorities" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "certAuthorityName": "my-ca",
    "certContent": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----"
  }'

# 3. Configure certificate matching
curl -X PATCH "http://broker:8080/SEMP/v2/config/msgVpns/default" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "authenticationClientCertUsernameSource": "common-name"
  }'
```

### OAuth 2.0 Authentication

```bash
# 1. Create OAuth profile
curl -X POST "http://broker:8080/SEMP/v2/config/oauthProfiles" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "oauthProfileName": "azure-ad",
    "clientId": "client-id-from-azure",
    "clientSecret": "client-secret",
    "issuerIdentifier": "https://login.microsoftonline.com/tenant-id/v2.0",
    "tokenIntrospectionUri": "https://login.microsoftonline.com/tenant-id/oauth2/v2.0/introspect",
    "enabled": true
  }'

# 2. Enable OAuth on VPN
curl -X PATCH "http://broker:8080/SEMP/v2/config/msgVpns/default" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "authenticationOauthEnabled": true,
    "authenticationOauthDefaultProviderName": "azure-ad"
  }'
```

### LDAP Authentication

```bash
# 1. Configure LDAP profile
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/authenticationLdapProfiles" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "ldapProfileName": "corporate-ldap",
    "serverPriority": 1,
    "serverHost": "ldap.example.com",
    "serverPort": 636,
    "adminDn": "cn=admin,dc=example,dc=com",
    "adminPassword": "ldap-admin-password",
    "searchBaseDn": "ou=users,dc=example,dc=com",
    "searchFilter": "(uid=${user})",
    "searchScope": "subtree",
    "tlsEnabled": true
  }'

# 2. Enable LDAP authentication
curl -X PATCH "http://broker:8080/SEMP/v2/config/msgVpns/default" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "authenticationBasicType": "ldap"
  }'
```

## Authorization (ACL Profiles)

### Create ACL Profile

```bash
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/aclProfiles" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "aclProfileName": "order-service-acl",
    "clientConnectDefaultAction": "allow",
    "publishTopicDefaultAction": "disallow",
    "subscribeTopicDefaultAction": "disallow"
  }'
```

### Topic Publish Exceptions

```bash
# Allow publishing to specific topics
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/aclProfiles/order-service-acl/publishTopicExceptions" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "publishTopicException": "acme/orders/>",
    "publishTopicExceptionSyntax": "smf"
  }'
```

### Topic Subscribe Exceptions

```bash
# Allow subscribing to specific topics
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/aclProfiles/order-service-acl/subscribeTopicExceptions" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "subscribeTopicException": "acme/inventory/>",
    "subscribeTopicExceptionSyntax": "smf"
  }'
```

### Example ACL Patterns

| Service | Publish | Subscribe |
|---------|---------|-----------|
| order-service | `acme/orders/>` | `acme/inventory/>`, `acme/payments/results/>` |
| inventory-service | `acme/inventory/>` | `acme/orders/>` |
| payment-service | `acme/payments/>` | `acme/orders/payment-required/>` |
| analytics | (none) | `acme/>` |

## TLS Configuration

### Server Certificate Setup

```bash
# 1. Upload server certificate
curl -X POST "http://broker:8080/SEMP/v2/config/serverCertificates" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "certContent": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "privateKeyContent": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----"
  }'

# 2. Enable TLS on VPN
curl -X PATCH "http://broker:8080/SEMP/v2/config/msgVpns/default" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "serviceTlsEnabled": true,
    "serviceTlsListenPort": 55443
  }'
```

### TLS Cipher Suites

```bash
# Configure strong cipher suites
curl -X PATCH "http://broker:8080/SEMP/v2/config" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "tlsCipherSuiteList": "ECDHE-RSA-AES256-GCM-SHA384,ECDHE-RSA-AES128-GCM-SHA256"
  }'
```

### Client TLS Configuration (Python)

```python
import ssl
import solace.messaging.messaging_service as messaging

ssl_context = ssl.create_default_context()
ssl_context.load_cert_chain(
    certfile='/path/to/client.crt',
    keyfile='/path/to/client.key'
)
ssl_context.load_verify_locations('/path/to/ca.crt')

service = messaging.MessagingService.builder() \
    .from_properties({
        "solace.messaging.transport.host": "tcps://broker:55443",
        "solace.messaging.service.vpn-name": "default"
    }) \
    .with_transport_security_strategy(
        messaging.TLS.create()
            .with_certificate_validation(True)
            .with_trusted_certificates('/path/to/ca.crt')
            .with_client_certificate('/path/to/client.crt', '/path/to/client.key')
    ) \
    .build()
```

## Queue-Level Access Control

### Restrict Queue Access

```bash
# Set queue owner
curl -X PATCH "http://broker:8080/SEMP/v2/config/msgVpns/default/queues/sensitive-queue" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "owner": "order-service",
    "permission": "no-access"
  }'

# Grant specific access
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/queues/sensitive-queue/subscriptions" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "subscriptionTopic": "acme/orders/sensitive/>",
    "subscriptionPermission": "consume"
  }'
```

## Audit Logging

### Enable Event Logging

```bash
# Enable client events
curl -X PATCH "http://broker:8080/SEMP/v2/config/msgVpns/default" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "eventClientConnectEnabled": true,
    "eventClientDisconnectEnabled": true,
    "eventSubscriptionMode": "off-with-warning"
  }'

# Enable system-level events
curl -X PATCH "http://broker:8080/SEMP/v2/config" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "eventPublishClientEnabled": true,
    "eventPublishMsgVpnEnabled": true
  }'
```

### Syslog Integration

```bash
# Configure syslog export
curl -X POST "http://broker:8080/SEMP/v2/config/msgVpns/default/syslogServers" \
  -u admin:admin \
  -H "Content-Type: application/json" \
  -d '{
    "syslogServerName": "security-siem",
    "host": "siem.example.com",
    "port": 514,
    "protocol": "tcp",
    "enabled": true
  }'
```

## Security Best Practices

### Password Policy

```bash
# Minimum password requirements
# Note: Configure at creation time
{
  "password": "ComplexP@ssw0rd!2024",  # Min 12 chars, mixed case, numbers, symbols
}
```

### Network Segmentation

```
┌─────────────────────────────────────────────────────┐
│                    DMZ / Edge                       │
│  ┌──────────────┐                                   │
│  │ MQTT Gateway │ ← IoT Devices (TLS)              │
│  └──────────────┘                                   │
└─────────────────────────────────────────────────────┘
         │ Internal only
         ▼
┌─────────────────────────────────────────────────────┐
│                 Internal Network                    │
│  ┌──────────────┐                                   │
│  │ Core Broker  │ ← Microservices (mTLS)           │
│  └──────────────┘                                   │
└─────────────────────────────────────────────────────┘
```

### Security Checklist

```markdown
## Pre-Production Security Checklist

### Authentication
- [ ] Default admin password changed
- [ ] Strong password policy enforced
- [ ] Client certificate authentication for services
- [ ] OAuth/LDAP for user access
- [ ] Service accounts have minimal privileges

### Authorization
- [ ] ACL profiles defined per service
- [ ] Default deny for publish/subscribe
- [ ] Queue ownership configured
- [ ] No wildcard (>) subscriptions in production

### Encryption
- [ ] TLS enabled for all client connections
- [ ] TLS 1.2+ only, weak ciphers disabled
- [ ] Internal broker communication encrypted
- [ ] Encryption at rest enabled (if available)

### Network
- [ ] SEMP API restricted to management network
- [ ] Firewall rules limit broker ports
- [ ] Health check endpoints not publicly exposed

### Audit
- [ ] Client connection events logged
- [ ] Admin operations logged
- [ ] Syslog/SIEM integration configured
- [ ] Log retention policy defined
```

### Rotating Credentials

```python
#!/usr/bin/env python3
"""Rotate client credentials without downtime"""
import requests
from requests.auth import HTTPBasicAuth

class CredentialRotator:
    def __init__(self, host, admin_user, admin_pass):
        self.base_url = f"http://{host}:8080/SEMP/v2/config"
        self.auth = HTTPBasicAuth(admin_user, admin_pass)
    
    def rotate_password(self, vpn, username, new_password):
        # 1. Create temporary user with new password
        temp_user = f"{username}-rotate"
        self._create_temp_user(vpn, temp_user, new_password, username)
        
        # 2. Update original user password
        self._update_password(vpn, username, new_password)
        
        # 3. Delete temporary user
        self._delete_user(vpn, temp_user)
        
        return True
    
    def _create_temp_user(self, vpn, temp_user, password, source_user):
        # Copy ACL and profile from source user
        source = self._get_user(vpn, source_user)
        
        url = f"{self.base_url}/msgVpns/{vpn}/clientUsernames"
        requests.post(url, json={
            "clientUsername": temp_user,
            "enabled": True,
            "password": password,
            "aclProfileName": source["aclProfileName"],
            "clientProfileName": source["clientProfileName"]
        }, auth=self.auth)
    
    def _update_password(self, vpn, username, password):
        url = f"{self.base_url}/msgVpns/{vpn}/clientUsernames/{username}"
        requests.patch(url, json={"password": password}, auth=self.auth)
```

## Reference Links

- **Security Overview**: https://docs.solace.com/Security/Security-Index.htm
- **Authentication**: https://docs.solace.com/Security/Client-Authentication.htm
- **ACL Profiles**: https://docs.solace.com/Security/ACL-Profiles.htm
- **TLS/SSL**: https://docs.solace.com/Security/TLS-SSL-Service-Connections.htm

## Further Reading

- [OAuth Patterns](./references/oauth.md)
- [Certificate Management](./references/certificates.md)
- [Compliance Guide](./references/compliance.md)
