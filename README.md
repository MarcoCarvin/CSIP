# CSIP Authentication System

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Status](https://img.shields.io/badge/status-prototype-orange.svg)

## Bridging Modern Identity with Linux Authentication

CSIP Authentication System provides a novel approach to Linux authentication by bridging modern identity providers (IdPs) with Linux's PAM and NSS systems, all while maintaining a **shadowless user model**.

## What Makes This Different?

Unlike traditional LDAP+NFS solutions or off-the-shelf MFA products, CSIP Authentication introduces:

- **Proxy-based authentication architecture** centralizing all authentication flows
- **Dynamic IdP discovery and heuristic user registration** without configuration changes
- **Multi-IdP support** with seamless integration of various identity providers through Keycloak
- **True shadowless users** with deterministic UID/GID generation (consistent across systems)
- **Browser-based authentication flows for SSH access** without requiring custom clients
- **Native WebAuthn/FIDO2 support** for hardware security keys with SSH
- **Direct OAuth/OIDC integration** with Linux's PAM/NSS stack
- **Dynamic role-based access control** tied directly to IdP roles
- **Mandatory MFA for sudo commands** to protect privileged operations

## Core Components

The system consists of three main components:

1. **PAM Module** (`pam_keycloak.so`) - Handles the authentication flow, deployed locally on each Linux system
2. **NSS Module** (`libnss_keycloak.so`) - Provides shadowless user lookup, deployed locally on each Linux system
3. **Authentication Proxy/Backend** - Centralized service that mediates between Linux systems and identity providers

## Architecture

### Centralized Proxy Architecture

CSIP Authentication uses a proxy-based architecture that provides critical advantages:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Linux      │     │ PAM/NSS     │     │ Auth Proxy  │     │  Keycloak   │
│  Systems    │────>│ Modules     │────>│ Service     │────>│  Server     │
└─────────────┘     └─────────────┘     └──────┬──────┘     └────┬────────┘
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



*Figure 1: Centralized proxy architecture showing integration between Linux systems, proxy service, Keycloak, and external identity providers*

## SIEM Integration

The authentication proxy service includes robust integration with Security Information and Event Management (SIEM) systems:

### Features
- **Comprehensive Authentication Logging**: Every authentication event, including logins, sudo commands, and MFA operations are captured
- **Standardized Event Format**: Events follow standardized formats (Common Event Format, Syslog, JSON) for easy integration
- **Real-time Streaming**: Events are streamed in real-time to SIEM platforms with minimal latency
- **Correlation IDs**: Each authentication flow receives a unique identifier for cross-system event correlation
- **Enriched Context**: Authentication events include contextual information about source, destination, and authentication method

### Security Analysis Capabilities
- Detection of authentication anomalies
- Brute force attempt monitoring
- Authentication pattern analysis
- Privileged account monitoring

Every authentication action - whether initial login or a sudo command - is routed through the centralized proxy service, creating a comprehensive security boundary and audit trail.

### Multi-IdP Integration

A powerful feature of CSIP Authentication is its ability to integrate with multiple identity providers simultaneously through Keycloak:

- **Dynamic IdP Discovery**: System automatically detects and presents the appropriate IdP based on username/email patterns
- **Zero-Configuration Provider Selection**: No need to modify configuration when adding new identity providers
- **Federated Identity Management**: Allow users to authenticate with their preferred provider
- **Heuristic User Registration**: New users are automatically directed to the appropriate IdP based on email domain or other attributes
- **Consistent User Experience**: Same Linux login experience regardless of backend identity source
- **Mandatory MFA**: All users must configure MFA during first login or registration before access is granted

During authentication, users are automatically directed to the appropriate identity provider:

```
$ ssh user@server
Authenticating...
[System detects user@company.com pattern]
Redirecting to Company SSO...
Authenticating with Company SSO...
Enter your TOTP code: 123456
Authentication successful

$ ssh newuser@server
New user detected.
Email address (for IdP selection): newuser@partner.org
[System detects partner.org domain and routes to appropriate IdP]
Redirecting to Partner SSO for account creation...
Please open https://auth.example.com/register/xyz789 to complete registration
[User completes registration and MFA setup]
MFA setup required for first login.
Scan this QR code with your authenticator app: [QR code displayed]
Enter the TOTP code to confirm setup: 654321
MFA configured successfully
```

This allows organizations to:
- Support users from multiple sources (employees, contractors, partners) without configuration changes
- Enable self-service registration for new users based on their organizational affiliation
- Implement a gradual migration between identity providers
- Leverage existing identity investments while standardizing Linux authentication
- Provide fallback authentication paths for high availability

## How It Works

### Authentication Flow

When a user attempts SSH access:

1. SSH prompts for username/password
2. PAM module validates credentials against the identity provider
3. For MFA, a browser link is provided (QR code in future versions)
4. User completes MFA in browser (TOTP or WebAuthn)
5. PAM module completes authentication
6. NSS module resolves the user identity without local accounts

### Shadowless User Model

Unlike traditional systems that require creating local accounts:

- User accounts exist only in the identity provider
- NSS module dynamically resolves user information at runtime
- Deterministic algorithm ensures UIDs/GIDs are consistent across systems
- Home directories are automatically created on first login (no NFS required)

### Authentication Flow Options

The system supports multiple MFA methods for terminal sessions:

#### 1. Browserless TOTP Authentication

TOTP (Time-based One-Time Password) authentication can be completed entirely within the terminal:

![TOTP Authentication Flow](images/mfa_flow_totp.png)

*Figure 2: TOTP authentication flow allowing verification directly within the terminal*

#### 2. WebAuthn/FIDO2 Browser-Based Authentication

For hardware security keys and biometrics, a browser-based flow is used:

![WebAuthn Authentication Flow](images/mfa_flow_webauthn.png)

*Figure 3: WebAuthn/FIDO2 authentication flow enabling hardware security key usage*

These approaches allow secure multi-factor authentication without requiring modifications to SSH clients, while supporting both browserless (TOTP) and browser-based (WebAuthn) authentication methods.

### Privileged Access Protection

A standout security feature of CSIP Authentication is mandatory MFA for each sudo command, providing enhanced protection for privileged operations:

- **Every sudo command requires MFA verification** - not just the first one in a session
- **Same authentication methods as login** - TOTP, WebAuthn/FIDO2 security keys, fingerprint readers
- **Time-limited verification** - configurable window before re-verification is required
- **Audit trail of privileged commands** - each verification is logged with the associated command

This approach significantly raises the security bar for privileged access:

![Sudo Authentication Flow](images/sudo_authentication_flow.png)

*Figure 4: Sudo command authentication flow showing MFA verification requirement*

By requiring MFA for each privileged command, the system provides strong protection against:
- Unattended terminal exploitation
- Credential theft via keylogging or over-the-shoulder attacks
- Accidental execution of dangerous commands

## Technical Tradeoffs

### Advantages

- Zero local user management
- Self-service user registration with automatic IdP routing
- Zero-configuration IdP integration based on user attributes
- Centralized authentication control and audit trail
- Multiple identity provider support through Keycloak
- Identity source flexibility and migration capabilities
- Complete visibility into all authentication attempts (login and sudo)
- Simplified security monitoring and threat detection
- Consistent permissions across systems
- Support for modern authentication methods
- Fine-grained security for privileged operations
- Easier compliance with security standards through central logging

### Current Limitations

- Manual browser interaction required for MFA (future versions will improve this)
- Performance impact of dynamic user resolution (mitigated by caching)
- Requires backend proxy connectivity for authentication
- Limited offline capability
- Single point of failure without proper proxy high-availability setup

## Project Status

⚠️ **PROTOTYPE / DEMONSTRATION** ⚠️

This project is currently a working prototype demonstrating the core concepts. While functional, it's not recommended for production use without thorough security review and hardening.

## Requirements

- Linux system with PAM and NSS support
- Keycloak instance (or compatible OIDC provider)
- Keycloak TOTP extension installed
- Keycloak webauthn extension installed
- Node.js runtime for the backend service
- PostgreSQL database


## Future Development

- [ ] Browser widget for smoother MFA experience
- [ ] CLI-based FIDO2 support that doesn't require browser
- [ ] Performance optimizations for large deployments

## Contributing

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

Areas where we particularly need help:
- Security auditing and hardening
- Performance optimization
- Documentation improvements
- Browser widget development

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- The Linux-PAM team for their documentation
- The WebAuthn and FIDO Alliance for standards development
- The Keycloak community for their excellent IdP implementation

---

### Interactive API Documentation

For detailed API specifications and interactive testing, see our [API Documentation](https://marcocarvin.github.io/CSIP/api/).

---

⚠️ **Security Notice**: This is authentication code. Please review thoroughly before deploying in any environment. The creators are not responsible for security vulnerabilities that may arise from improper deployment or configuration.
