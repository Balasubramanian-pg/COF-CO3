# Organization and Account Objects in Snowflake

```mermaid
graph TD
  Org[Organization] --> Acc1[Account 1]
  Org --> Acc2[Account 2]
  Org --> Acc3[Account 3]
  
  Acc1 --> DB1[Database]
  Acc1 --> WH1[Warehouse]
  Acc1 --> Role1[Roles]
  
  Acc2 --> DB2[Database]
  Acc2 --> WH2[Warehouse]
  
  Acc3 --> DB3[Database]
```

| Level | What It Is | What You Manage Here |
|-------|-----------|---------------------|
| Organization | Top level container for multiple accounts | Organization wide policies, account provisioning, SSO setup, billing aggregation |
| Account | Isolated Snowflake environment | Databases, warehouses, users, roles, shares, tasks |

```mermaid
flowchart LR
  A[Create Organization] --> B[Add Accounts to Org]
  B --> C[Set Org Level Policies]
  C --> D[Manage Each Account Separately]
  D --> E[View Unified Billing]
```

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/af7c1481-f3db-4c72-aced-06d2cda2e791" />

## Organization Level Objects

- Organization account is The master account that creates and manages other accounts
- Account listings: View all accounts under the organization with status and region
- Organization roles: Roles that span multiple accounts for cross account administration
- Policy templates: Define security, network, or data policies to apply across accounts
- Billing view: Consolidated credit usage and cost reporting for all accounts

## Account Level Objects

- Users: Individual people or service accounts that log in
- Roles: Collections of privileges assigned to users or other roles
- Warehouses: Compute clusters that run queries and load data
- Databases: Logical containers for schemas and tables
- Integrations: Connections to external systems like S3, Azure Blob, or OAuth providers
- Shares: Secure data sharing configurations for internal or external consumers

```mermaid
quadrantChart
  title "When to Use Multi Account Setup"
  x-axis "Single Team" --> "Multiple Teams"
  y-axis "Simple Needs" --> "Complex Governance"
  "Small startup": [0.2, 0.2]
  "Single project": [0.3, 0.3]
  "Dev test prod separation": [0.7, 0.5]
  "Regulated multi region": [0.9, 0.9]
```

| Scenario | Recommended Setup | Why |
|----------|------------------|-----|
| One team, one project | Single account | Less overhead, simpler management |
| Dev, test, prod environments | Separate accounts per environment | Isolate risk, control costs, prevent accidental changes |
| Multiple business units | One account per unit | Clear ownership, separate billing, independent scaling |
| Compliance boundaries | Separate accounts per regulation | Enforce different policies, limit data exposure |
| Multi region deployment | Accounts in each region | Reduce latency, meet data residency rules |

```mermaid
flowchart TD
  Q1[Start: How many teams or environments]
  Q1 --> Q2[One team, one environment]
  Q1 --> Q3[Multiple environments or teams]
  
  Q2 --> A[Single Account]
  Q3 --> Q4[Need isolation between environments]
  
  Q4 -->|Yes| B[Multiple Accounts]
  Q4 -->|No| C[Single Account with Role Separation]
  
  B --> D[Use Organization to Manage All]
  C --> E[Use Roles and Warehouses to Separate Work]
```

## Account Creation and Management

| Action | Where You Do It | What Happens |
|--------|----------------|--------------|
| Create new account | Organization admin console | New isolated Snowflake account with admin user |
| Suspend account | Organization console or API | Account becomes read only, no new queries allowed |
| Resume account | Organization console | Account returns to full operation |
| Delete account | Organization console with confirmation | Account and all data permanently removed after retention period |
| Transfer account | Organization admin | Move account from one organization to another |

## Security and Governance Boundaries

- Users and roles exist inside one account only. They do not automatically work in other accounts
- Data sharing can cross account boundaries using secure shares, but objects stay in source account
- Warehouses cannot be shared. Each account manages its own compute resources
- Organization level policies can enforce rules like MFA or IP allow lists across all accounts
- Audit logs are per account. Organization view aggregates but does not merge detailed logs

```mermaid
sequenceDiagram
  participant OrgAdmin as Organization Admin
  participant OrgSys as Snowflake Org Service
  participant NewAcc as New Account
  
  OrgAdmin->>OrgSys: Request new account with name and region
  OrgSys->>NewAcc: Provision isolated environment
  OrgSys->>OrgAdmin: Return account locator and admin credentials
  OrgAdmin->>NewAcc: Log in and configure databases, roles, warehouses
  NewAcc->>OrgSys: Report usage and billing to organization
```

## Common Setup Patterns

| Pattern | Structure | Best For |
|---------|-----------|----------|
| Environment isolation | dev_account, test_account, prod_account | Teams that need safe promotion pipelines |
| Business unit separation | finance_account, marketing_account, ops_account | Large organizations with independent budgets |
| Regional compliance | us_east_account, eu_west_account, ap_south_account | Data residency requirements or latency optimization |
| Project based | project_alpha_account, project_beta_account | Short term initiatives with clear end dates |
| Hybrid | prod_account + shared_dev_account | Balance isolation with resource efficiency |

## Things to Watch For

- Account locators are permanent. Choose naming carefully before creation
- Changing an account name does not change its locator. Applications connect via locator
- Cross account queries require secure shares or replication. They do not work automatically
- Billing rolls up to organization, but cost allocation tags must be set per account
- Organization admins can access any account. Limit this role to trusted personnel only
- Replication and failover work between accounts but require explicit setup and permissions

```mermaid
flowchart TD
  Q1[Start: What drives your structure]
  Q1 --> Q2[Team or project boundaries]
  Q1 --> Q3[Compliance or region rules]
  Q1 --> Q4[Cost tracking needs]
  
  Q2 --> A[Create account per team or project]
  Q3 --> B[Create account per region or regulation]
  Q4 --> C[Use tags and warehouses for cost split, or separate accounts]
  
  A --> D[Use Organization for unified management]
  B --> D
  C --> D
```

## Key Points

- Organization is for management. Account is for work. Keep that separation clear
- Start with one account if you are small. Split later when isolation becomes valuable
- Use roles and warehouses to separate work inside an account before adding more accounts
- Document your account naming convention early. It is hard to change later
- Test cross account sharing before you depend on it in production
- Monitor credit usage at both account and organization level to catch surprises early

Think of organization and accounts like a company and its departments:
- Organization sets the rules and pays the bills
- Each account runs its own work with its own tools
- Departments can share information when needed, but stay independent
- Adding a new department is easy. Merging them later is hard

Design your structure for how you work today, with room to grow tomorrow.
