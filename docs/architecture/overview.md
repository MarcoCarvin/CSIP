# CSIP Authentication System: Architecture Overview

## Introduction

The CSIP Authentication System provides a novel approach to Linux authentication by integrating modern identity providers (IdPs) with traditional Linux authentication mechanisms (PAM and NSS), all while eliminating the need for local shadow password files. This document provides a high-level overview of the system architecture, components, and key design principles.

## Core Design Principles

The CSIP Authentication System is built on several key principles:

1. **Centralized Authentication Control**: All authentication decisions flow through a central proxy service.
2. **Shadowless User Model**: Users exist only in the identity provider, not as local accounts.
3. **Multi-IdP Support**: Seamless integration with multiple identity providers simultaneously.
4. **Dynamic User Resolution**: User information is resolved at runtime without requiring local accounts.
5. **Enhanced Security for Privileged Access**: Mandatory MFA for sudo commands.
6. **Comprehensive Audit Trail**: All authentication events are logged and available for security analysis.

## System Components

The architecture consists of three primary components:

### 1. PAM Module (`pam_keycloak.so`)

- **Deployed on**: Each Linux system requiring authentication
- **Responsibilities**:
  - Intercept authentication requests (login, sudo, etc.)
  - Communicate with the authentication proxy
  - Manage the MFA flow (terminal or browser-based)
  - Enforce authentication decisions

### 2. NSS Module (`libnss_keycloak.so`) 

- **Deployed on**: Each Linux system requiring authentication
- **Responsibilities**:
  - Resolve user information dynamically without local accounts
  - Generate consistent UIDs/GIDs across systems
  - Create home directories on first login
  - Cache user information for performance

### 3. Authentication Proxy/Backend

- **Deployed as**: Centralized service (high availability recommended)
- **Responsibilities**:
  - Mediate between Linux systems and identity providers
  - Route authentication requests to appropriate IdPs
  - Handle dynamic IdP discovery
  - Manage MFA flows
  - Create comprehensive audit logs
  - Integrate with SIEM platforms

## Architecture Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Linux      │     │ PAM/NSS     │     │ Auth Proxy  │     │  Keycloak   │
│  Systems    │────>│ Modules     │────>│ Service     │────>│  Server     │
└─────────────┘     └─────────────┘     └──────┬──────┘     └──────┬──────┘
     Many               Local              Centralized           │
                                               │                 │
                                        ┌──────┘                 │
                                        ▼                        │
                                   ┌────────────┐                │
                                   │   SIEM     │                │
                                   │  Platform  │                │
                                   └────────────┘                │
                                                                 │
                                                          ┌──────┴──────┐
                                                          │             │
                                                     ┌────┴─────┐ ┌─────┴────┐
                                                     │ Active   │ │ Other    │
                                                     │ Directory│ │ IdPs     │
                                                     └──────────┘ └──────────┘
```

## Authentication Flow

When a user attempts to authenticate:

1. The PAM module intercepts the authentication request
2. The request is forwarded to the authentication proxy
3. The proxy identifies the appropriate IdP based on username/email patterns
4. The user completes authentication with their IdP
5. For MFA, either:
   - TOTP is entered directly in the terminal, or
   - A browser link is provided for WebAuthn/FIDO2 authentication
6. The authentication result is returned to the PAM module
7. If successful, the NSS module resolves the user information
8. The user is granted access

## Shadowless User Model

Unlike traditional Linux authentication:

- Users exist only in the identity provider
- No local `/etc/passwd` or `/etc/shadow` entries are created
- User information is resolved dynamically at runtime
- A deterministic algorithm ensures consistent UIDs/GIDs across systems
- Home directories are created automatically on first login

This approach eliminates local user management and provides a centralized identity source.

## Multi-IdP Integration

The system leverages Keycloak to integrate with multiple identity providers:

- Dynamic IdP discovery based on username/email patterns
- Zero-configuration provider selection
- Support for Active Directory, SAML, OIDC, and other identity sources
- Consistent user experience regardless of backend identity provider
- Self-service user registration with automatic IdP routing

## Privileged Access Protection

A key security feature is mandatory MFA for sudo commands:

- Every sudo command requires MFA verification
- Multiple authentication methods supported (TOTP, WebAuthn/FIDO2)
- Time-limited verification window
- Comprehensive audit trail of privileged commands

## SIEM Integration

The authentication proxy includes comprehensive logging and SIEM integration:

- All authentication events are logged in standardized formats
- Real-time event streaming to SIEM platforms
- Correlation IDs for cross-system event tracking
- Enriched context for security analysis
- Support for authentication anomaly detection

## Technical Considerations

### Security Boundaries

The centralized proxy architecture creates a clear security boundary:

- All authentication decisions pass through a single point
- Authentication logic is isolated from the individual systems
- Consistent security policies can be enforced across all systems
- Authentication events are visible and auditable

### High Availability

For production deployments:

- The authentication proxy should be deployed in a high-availability configuration
- Multiple proxy instances behind a load balancer
- Database replication for user caching
- Fallback authentication mechanisms for outage scenarios

### Network Requirements

The system requires:

- Linux systems must be able to reach the authentication proxy
- The proxy must be able to reach the Keycloak server
- Users must be able to reach the MFA verification URLs (for browser-based MFA)

## Conclusion

The CSIP Authentication System provides a modern approach to Linux authentication that bridges the gap between traditional Linux systems and modern identity management. By centralizing authentication decisions, eliminating local user management, and supporting multiple identity providers, it simplifies administration while enhancing security through mandatory MFA and comprehensive audit logging.