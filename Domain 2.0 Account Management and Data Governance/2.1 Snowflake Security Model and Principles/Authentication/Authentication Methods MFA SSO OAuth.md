# Authentication Methods: MFA SSO OAuth in Snowflake

```mermaid
graph TD
  Auth[Authentication Methods] --> MFA[Multi Factor Authentication]
  Auth --> SSO[Single Sign On SAML]
  Auth --> OAuth[OAuth 2.0]
  Auth --> KeyPair[Key Pair Authentication]
  Auth --> Password[Username and Password]
```

## Authentication Method Comparison

| Method | Setup Effort | Security Level | Best For | Not Good For |
|--------|-------------|---------------|----------|--------------|
| Username and Password | Low | Medium | Learning personal use quick tests | Production automation service accounts |
| Key Pair | Medium | High | ETL pipelines scripts service accounts | Interactive human users |
| OAuth 2.0 | High | High | Web apps mobile apps API access | One off scripts ad hoc queries |
| SSO with SAML | High | High | Corporate employees with IdP | External users without IdP access |
| External Browser | Low | Medium | Interactive CLI or notebook use | Headless automation scheduled jobs |

```mermaid
flowchart LR
  Q1[Start: Who needs to log in]
  Q1 --> Q2[Human user]
  Q1 --> Q3[Service or script]
  
  Q2 --> Q4[Does org use SSO]
  Q3 --> Q5[Needs long lived access]
  
  Q4 -->|Yes| A[Use SSO with SAML]
  Q4 -->|No| B[Use password or external browser]
  
  Q5 -->|Yes| C[Use key pair authentication]
  Q5 -->|No| D[Use OAuth with short tokens]
```

## Multi Factor Authentication MFA

### What MFA Does

| Concept | Simple Explanation |
|---------|------------------|
| First factor | Something you know like a password |
| Second factor | Something you have like a phone or token |
| MFA | Requires both factors to log in |
| MFA caching | Remembering MFA for a time window |

### MFA Methods in Snowflake

| Method | How It Works | Security Level | User Experience |
|--------|-------------|---------------|----------------|
| Duo Push | Push notification to phone app | High | Tap approve on phone |
| Okta Verify | App notification or code | High | Open app and approve |
| Google Authenticator | Time based code from app | Medium High | Type 6 digit code |
| SMS code | Text message with code | Medium | Type code from text |
| Email code | Email with verification code | Low Medium | Type code from email |

```mermaid
sequenceDiagram
  participant User as Human User
  participant SF as Snowflake
  participant MFA as MFA Provider
  
  User->>SF: Enter username and password
  SF->>MFA: Request second factor
  MFA->>User: Send push or code
  User->>MFA: Approve or enter code
  MFA-->>SF: Confirm second factor
  SF-->>User: Grant access
```

### Enforcing MFA

```sql
-- Require MFA for specific user
ALTER USER analyst_jane SET MFA_ENROLLMENT = REQUIRED;

-- Disable MFA caching for sensitive roles
ALTER ACCOUNT SET ALLOW_CLIENT_MFA_CACHING = FALSE;

-- Check MFA status for all users
SELECT name, mfa_enrollment, last_success_login
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE mfa_enrollment IS NOT NULL;
```

### MFA Best Practices

- Require MFA for all interactive users. Passwords alone are not enough
- Use app based MFA not SMS. SMS can be intercepted via SIM swap attacks
- Do not cache MFA for admin roles. Force re authentication for privileged actions
- Monitor MFA failures. Look for brute force patterns in LOGIN_HISTORY
- Document recovery steps. Users need a path if they lose their MFA device

```mermaid
flowchart TD
  Q1[Start: Evaluate MFA need]
  Q1 --> Q2[Is this an interactive user]
  Q2 -->|No| A[MFA not applicable]
  Q2 -->|Yes| Q3[Does role have elevated privileges]
  
  Q3 -->|Yes| B[Require MFA no caching]
  Q3 -->|No| Q4[Is data sensitive or regulated]
  
  Q4 -->|Yes| B
  Q4 -->|No| C[MFA recommended but optional]
  
  B --> D[Use app based MFA]
  C --> E[Allow SMS or email fallback]
```

## Single Sign On SSO with SAML

### What SSO Does

| Concept | Simple Explanation |
|---------|------------------|
| Identity Provider IdP | Central system that verifies user identity |
| SAML | Standard protocol for sharing authentication |
| SSO | Log in once access many systems |
| Assertion | Signed message from IdP confirming identity |

### How SSO Flow Works

```mermaid
sequenceDiagram
  participant User as Human User
  participant Browser as Web Browser
  participant SF as Snowflake
  participant IdP as Identity Provider
  
  User->>Browser: Go to Snowflake login
  Browser->>SF: Request authentication
  SF->>Browser: Redirect to IdP with SAML request
  Browser->>IdP: Present SAML request
  IdP->>User: Prompt for credentials and MFA
  User->>IdP: Authenticate successfully
  IdP->>Browser: Return signed SAML assertion
  Browser->>SF: Present SAML assertion
  SF->>SF: Validate signature and extract user
  SF-->>Browser: Issue Snowflake session
  Browser-->>User: Show Snowsight dashboard
```

### Configuration Steps

```sql
-- Create SAML security integration
CREATE OR REPLACE SECURITY INTEGRATION my_sso_integration
  TYPE = SAML2
  ENABLED = TRUE
  SAML2_ISSUER = 'https://idp.example.com/saml'
  SAML2_SSO_URL = 'https://idp.example.com/sso'
  SAML2_PROVIDER = 'CUSTOM'
  SAML2_X509_CERT = 'MIIDXTCCAkWgAwIBAgIJ...'
  SAML2_SP_ISSUER = 'snowflake_account';

-- Get Snowflake metadata for IdP configuration
SELECT SYSTEM$GENERATE_SAML_SP_METADATA('my_sso_integration');
```

### User and Role Mapping

| IdP Attribute | Snowflake Field | Example |
|--------------|----------------|---------|
| NameID | User login name | jane.doe@company.com |
| Email | User email | jane.doe@company.com |
| Groups | Roles to assign | analytics_team engineering_team |
| FirstName | User first name | Jane |
| LastName | User last name | Doe |

```sql
-- Example: IdP sends group analytics_team
-- Snowflake maps to role ANALYST_ROLE via SCIM provisioning

-- Check user attributes from SSO
SELECT name, email, displayed_name
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE authentication_method = 'SAML2';
```

### When To Use SSO

| Scenario | Use SSO | Why |
|----------|---------|-----|
| Corporate employees with IdP | Yes | Centralized identity and MFA |
| External contractors without IdP | No | They cannot authenticate via your IdP |
| Mixed internal and external users | Maybe | Use SSO for internal password for external |
| Compliance requiring MFA | Yes | MFA enforced at IdP level |
| Small team without IdP | No | Password or external browser is simpler |

```mermaid
flowchart TD
  Q1[Start: Evaluate SSO]
  Q1 --> Q2[Do users have corporate IdP access]
  Q2 -->|No| A[Use password or external browser]
  Q2 -->|Yes| Q3[Is MFA required by policy]
  
  Q3 -->|Yes| B[SSO with SAML is appropriate]
  Q3 -->|No| Q4[Do you want centralized user management]
  
  Q4 -->|Yes| B
  Q4 -->|No| C[Password auth may be sufficient]
```

## OAuth 2.0 Authentication

### What OAuth Does

| Concept | Simple Explanation |
|---------|------------------|
| Access token | Short lived credential for API calls |
| Refresh token | Long lived token to get new access tokens |
| Client credentials | App authenticates itself without user |
| Authorization code | User approves app access via redirect |

### OAuth Flow Types

| Flow | When To Use | Key Characteristic |
|------|-------------|-------------------|
| Authorization Code | User interactive web apps | Redirects user to IdP for login |
| Client Credentials | Service to service calls | App authenticates itself no user context |
| Refresh Token | Long running user sessions | Use refresh token to get new access token |
| JWT Bearer | Assertion based authentication | App signs JWT to prove identity |

### How OAuth Flow Works

```mermaid
sequenceDiagram
  participant App as Your Application
  participant IdP as Identity Provider
  participant SF as Snowflake
  
  App->>IdP: Request token with client credentials
  IdP->>App: Return access token and refresh token
  App->>SF: Connect with access token
  SF->>IdP: Validate token signature and claims
  IdP-->>SF: Confirm token is valid
  SF-->>App: Issue Snowflake session
  App->>IdP: Use refresh token when access expires
  IdP-->>App: Return new access token
```

### Configuration Steps

```sql
-- Create OAuth security integration
CREATE OR REPLACE SECURITY INTEGRATION my_oauth_integration
  TYPE = OAUTH
  ENABLED = TRUE
  OAUTH_CLIENT = CUSTOM
  OAUTH_REDIRECT_URI = 'https://your-app.com/callback'
  OAUTH_ISSUE_REFRESH_TOKENS = TRUE
  OAUTH_REFRESH_TOKEN_VALIDITY = 7776000;

-- Get client credentials for your app
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('my_oauth_integration');
```

### Connecting with OAuth

```python
# Python connector example
import snowflake.connector

conn = snowflake.connector.connect(
    user='api_user',
    account='myaccount',
    authenticator='oauth',
    token='your_access_token_here'
)

# Handle token refresh in production code
# When access token expires use refresh token to get new one
```

### When To Use OAuth

| Scenario | Use OAuth | Why |
|----------|-----------|-----|
| Web app with user login | Yes | Standard flow for browser based apps |
| Mobile app connecting to Snowflake | Yes | Handles token refresh automatically |
| Microservice calling Snowflake API | Yes | Client credentials flow for service auth |
| One off script or ad hoc query | No | Overhead not justified for simple use |
| Internal tool with few users | Maybe | Password or key pair may be simpler |

```mermaid
flowchart TD
  Q1[Start: Evaluate OAuth]
  Q1 --> Q2[Is this an app with users]
  Q2 -->|Yes| Q3[Do you need token refresh]
  Q2 -->|No| Q4[Is this service to service]
  
  Q3 -->|Yes| A[Use OAuth with refresh tokens]
  Q3 -->|No| B[Use OAuth with short lived tokens]
  
  Q4 -->|Yes| C[Use OAuth client credentials flow]
  Q4 -->|No| D[Consider key pair or password instead]
```

## Key Pair Authentication Quick Reference

### Setup Steps

```bash
# Generate RSA key pair
openssl genrsa -out rsa_key.pem 2048
openssl rsa -in rsa_key.pem -pubout -out rsa_key_pub.pem

# Extract public key for Snowflake remove headers
cat rsa_key_pub.pem | grep -v "BEGIN\|END" | tr -d '\n'

# Assign public key to user
ALTER USER etl_service SET RSA_PUBLIC_KEY = 'MIIBIjANBgkq...';

# Connect with private key
snowsql -a myaccount -u etl_service --private-key-path rsa_key.pem
```

### Key Rotation Pattern

```sql
-- Add new key while keeping old one active
ALTER USER etl_service SET RSA_PUBLIC_KEY_2 = 'NEW_PUBLIC_KEY_HERE';

-- Update clients to use new private key

-- Remove old key after confirmation
ALTER USER etl_service UNSET RSA_PUBLIC_KEY;
ALTER USER etl_service SET RSA_PUBLIC_KEY = RSA_PUBLIC_KEY_2;
ALTER USER etl_service UNSET RSA_PUBLIC_KEY_2;
```

| Practice | Why It Matters |
|----------|---------------|
| Use 2048 or 4096 bit keys | Shorter keys are vulnerable to brute force |
| Store private keys in vault | Never commit keys to code repos |
| Rotate keys every 90 days | Limits exposure if a key is compromised |
| Use RSA_PUBLIC_KEY_2 for rotation | Enables zero downtime key updates |
| Audit key usage with LOGIN_HISTORY | Detect unusual authentication patterns |

## External Browser Authentication

### How It Works

```mermaid
sequenceDiagram
  participant User as Human User
  participant Client as snowsql or Connector
  participant Browser as System Browser
  participant SF as Snowflake
  
  User->>Client: Run command with authenticator=externalbrowser
  Client->>Browser: Open Snowflake login page
  User->>Browser: Enter credentials and MFA
  Browser->>SF: Authenticate and get token
  SF->>Client: Return token to client
  Client->>User: Proceed with authenticated session
```

### Usage Examples

```bash
# With snowsql
snowsql -a myaccount -u analyst_jane --authenticator externalbrowser

# With Python connector
import snowflake.connector
conn = snowflake.connector.connect(
    user='analyst_jane',
    account='myaccount',
    authenticator='externalbrowser'
)
```

| Benefit | Why It Matters |
|---------|---------------|
| No password in config | Credentials entered interactively not stored |
| MFA support | Works with Snowflake native MFA |
| Simple setup | No IdP or key management required |
| Works with SSO | Can redirect to IdP if configured |

### When To Use External Browser

| Scenario | Use External Browser | Why |
|----------|---------------------|-----|
| Interactive CLI or notebook use | Yes | Simple secure login without config |
| Headless automation | No | Requires human interaction to authenticate |
| Users with MFA enabled | Yes | Supports MFA flow naturally |
| Quick testing or demos | Yes | Fast setup without complex config |
| Production scheduled jobs | No | Cannot run unattended |

## Monitoring and Auditing Authentication

### Query Login History

```sql
-- Find failed login attempts in last 24 hours
SELECT
  EVENT_TIMESTAMP,
  USER_NAME,
  CLIENT_IP,
  ERROR_MESSAGE,
  AUTHENTICATION_METHOD
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE SUCCESS = 'NO'
  AND EVENT_TIMESTAMP > DATEADD(hour, -24, CURRENT_TIMESTAMP)
ORDER BY EVENT_TIMESTAMP DESC;

-- Track MFA enrollment status
SELECT
  name as user_name,
  mfa_enrollment,
  last_success_login,
  disabled
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE mfa_enrollment IS NOT NULL;
```

### Key Metrics to Watch

| Metric | Where To Find | Alert Threshold |
|--------|--------------|-----------------|
| Failed login attempts | LOGIN_HISTORY | More than 5 failures per hour per user |
| MFA enrollment rate | USERS view | Less than 90 percent of interactive users |
| Unusual login locations | LOGIN_HISTORY CLIENT_IP | Logins from new countries or IP ranges |
| Service account login patterns | LOGIN_HISTORY filtered by user type | Logins outside expected schedule |
| Token refresh failures | Query history with OAuth errors | Spike in authentication errors |

## Common Pitfalls and Fixes

| Mistake | What Happens | How To Fix |
|---------|--------------|------------|
| Storing passwords in scripts | Credentials leak via version control or logs | Use key pair or OAuth for automation |
| Not rotating keys | Compromised key grants long term access | Set 90 day rotation schedule use RSA_PUBLIC_KEY_2 |
| Allowing MFA caching for admin roles | Stolen session token grants prolonged access | Set ALLOW_CLIENT_MFA_CACHING = FALSE for privileged roles |
| Using SMS MFA for sensitive accounts | SIM swap attacks bypass SMS | Require app based MFA like Duo or Google Authenticator |
| Not monitoring LOGIN_HISTORY | Breaches go undetected | Set up alerts for failed logins or unusual IPs |
| Configuring SSO without testing | Users locked out after cutover | Test with pilot group before full rollout |
| Forgetting token refresh in OAuth code | App stops working when token expires | Implement refresh token logic in authentication flow |

```mermaid
flowchart TD
  Prob[Auth issue] --> Q1[User cannot login]
  Prob --> Q2[Service account fails]
  Prob --> Q3[Suspicious login detected]
  
  Q1 --> A[Check password expiry and MFA enrollment]
  Q1 --> B[Verify SSO configuration and IdP status]
  
  Q2 --> C[Check key rotation and private key path]
  Q2 --> D[Verify OAuth token is not expired]
  
  Q3 --> E[Review LOGIN_HISTORY for IP and method]
  Q3 --> F[Disable user and rotate credentials if compromised]
  
  A --> G[Test with representative account]
  B --> G
  C --> G
  D --> G
  E --> G
  F --> G
```

## Decision Framework

```mermaid
flowchart TD
  Q1[Start: What needs authentication]
  Q1 --> Q2[Human interactive user]
  Q1 --> Q3[Service or automation]
  Q1 --> Q4[Web or mobile app]
  
  Q2 --> Q5[Does org have SSO IdP]
  Q5 -->|Yes| A[Use SSO with SAML enable MFA]
  Q5 -->|No| B[Use external browser with MFA]
  
  Q3 --> Q6[Needs long lived access]
  Q6 -->|Yes| C[Use key pair authentication]
  Q6 -->|No| D[Use OAuth client credentials]
  
  Q4 --> Q7[Has user login flow]
  Q7 -->|Yes| E[Use OAuth authorization code]
  Q7 -->|No| F[Use OAuth client credentials]
  
  A --> G[Monitor LOGIN_HISTORY]
  B --> G
  C --> H[Rotate keys quarterly]
  D --> I[Handle token refresh]
  E --> I
  F --> I
```

## Best Practices Summary

- Match auth method to user type. Humans need simple. Services need secure. Apps need standard
- Require MFA for all interactive users. App based MFA is more secure than SMS
- Use SSO for corporate users. Centralized identity reduces overhead and improves security
- Use key pair for service accounts. No passwords to manage keys can be rotated securely
- Use OAuth for apps. Standard protocol handles tokens and refresh automatically
- Rotate credentials regularly. Passwords every 90 days. Keys every 90 days. Tokens as designed
- Monitor authentication events. Detect brute force or unusual patterns early
- Restrict by network policy. Limit login sources to known trusted IPs
- Document authentication choices. Future you needs to know why a method was selected
- Test before rollout. Pilot with small group before full deployment

## Bottom Line

- Authentication is your first line of defense. Get it right or everything else is weaker
- MFA is not optional for interactive access. Stolen passwords are too common
- SSO simplifies login for corporate users. One identity many systems
- OAuth is the standard for apps. Handles tokens and refresh automatically
- Key pairs are best for automation. Secure scriptable rotatable
- Monitor and alert. You cannot fix what you do not see
- Rotate and review. Credentials age. Access patterns change

Think of authentication like keys to a building:
- Passwords are like simple keys. Easy to copy easy to lose
- MFA is like badge plus PIN. Two factors harder to fake
- SSO is like a master badge from your employer. One login many doors
- OAuth is like a temporary guest pass. Expires automatically
- Key pairs are like encrypted key cards. Secure and rotatable

Pick the right key for the right door. Simple keys for low risk areas. Badges and codes for sensitive rooms. Temporary passes for visitors. And always check who used which key and when. That is how authentication works in Snowflake.
