# Compare and Contrast Snowflake Editions

# Snowflake Editions Simple Comparison

## Architecture Overview

```mermaid
graph LR
    A[Standard] --> B[Enterprise]
    B --> C[Business Critical]
    C --> D[VPS]
```

Each edition includes everything from the edition below it plus additional features.


## Feature Comparison Table

| Feature | Standard | Enterprise | Business Critical | VPS |
|---------|----------|------------|-------------------|-----|
| Core SQL and data loading | Yes | Yes | Yes | Yes |
| Basic encryption and security | Yes | Yes | Yes | Yes |
| Time Travel data recovery | 1 day | 90 days | 90 days | 90 days |
| Multi-cluster warehouses | No | Yes | Yes | Yes |
| Row and column level security | No | Yes | Yes | Yes |
| Customer managed encryption keys | No | No | Yes | Yes |
| Private network connectivity | No | No | Yes | Yes |
| HIPAA and PHI compliance support | No | No | Yes | Yes |
| Dedicated isolated hardware | No | No | No | Yes |
| Automated disaster recovery failover | No | No | Yes | Yes |


## Edition Use Cases

### Standard Edition

```mermaid
quadrantChart
    title "Standard Edition Best Fit"
    x-axis "Low Complexity" --> "High Complexity"
    y-axis "Low Risk" --> "High Risk"
    "Startups": [0.2, 0.2]
    "Small teams": [0.3, 0.3]
    "Development and testing": [0.25, 0.25]
```

- You are evaluating or learning Snowflake
- Small team with straightforward data needs
- No regulatory or compliance requirements
- Cost efficiency is a priority

### Enterprise Edition

```mermaid
quadrantChart
    title "Enterprise Edition Best Fit"
    x-axis "Low Complexity" --> "High Complexity"
    y-axis "Low Risk" --> "High Risk"
    "Growing companies": [0.6, 0.4]
    "Data teams": [0.7, 0.5]
    "Analytics at scale": [0.65, 0.45]
```

- Multiple teams or workloads running concurrently
- Need longer data recovery window (90 days)
- Require fine grained access controls
- Want automatic scaling for variable demand

### Business Critical Edition

```mermaid
quadrantChart
    title "Business Critical Edition Best Fit"
    x-axis "Low Complexity" --> "High Complexity"
    y-axis "Low Risk" --> "High Risk"
    "Healthcare": [0.8, 0.9]
    "Financial services": [0.85, 0.85]
    "Government and regulated": [0.9, 0.95]
```

- Handling sensitive data like health records or financial information
- Must comply with HIPAA, PCI DSS, FedRAMP or similar
- Need to manage your own encryption keys
- Require high availability with automatic failover

### VPS Edition

```mermaid
quadrantChart
    title "VPS Edition Best Fit"
    x-axis "Shared Resources" --> "Fully Isolated"
    y-axis "Standard Security" --> "Maximum Isolation"
    "Large financial institutions": [0.95, 0.98]
    "Defense and intelligence": [0.98, 0.99]
    "Ultra sensitive workloads": [0.97, 0.97]
```

- Require physical separation from other Snowflake customers
- Highest level of security and compliance needed
- Budget is secondary to isolation and control
- Typically large organizations with strict governance


## Pricing Overview (US East, AWS)

| Edition | Approximate Price Per Credit | Typical Use Case |
|---------|-----------------------------|------------------|
| Standard | 2.00 USD | Learning, small projects, proof of concept |
| Enterprise | 3.00 USD | Most production workloads and growing teams |
| Business Critical | 4.00 USD | Regulated industries and sensitive data |
| VPS | Contact sales | Maximum isolation requirements |

Note: Credits represent compute resources. You are charged only for what you use.


## What Changes Between Editions

### Security and Control Progression

```mermaid
flowchart TD
    S[Standard] -->|adds| E[Enterprise]
    E -->|adds| BC[Business Critical]
    BC -->|adds| V[VPS]
    
    subgraph "Added in Enterprise"
        E1[90 day Time Travel]
        E2[Row and column masking]
        E3[Multi-cluster warehouse scaling]
    end
    
    subgraph "Added in Business Critical"
        BC1[Customer managed encryption keys]
        BC2[Private network connectivity]
        BC3[HIPAA and PHI compliance]
        BC4[Automated failover for disaster recovery]
    end
    
    subgraph "Added in VPS"
        V1[Dedicated hardware resources]
        V2[No resource sharing with other customers]
        V3[Maximum network and compute isolation]
    end
```

### Performance and Scaling

- Standard: Single warehouse execution. Suitable for predictable, simple workloads.
- Enterprise and above: Support multiple warehouses that can scale automatically based on demand.
- All editions use the same underlying query engine. Performance is not throttled on lower tiers.

### Data Recovery Capabilities

| Edition | Time Travel Recovery Window |
|---------|----------------------------|
| Standard | 24 hours |
| Enterprise and above | Up to 90 days |
| All editions | Additional 7 days in Fail-safe mode (emergency recovery only, read only) |


## Common Selection Errors

- Assuming you can easily switch editions later without planning. While upgrading is simple, moving sensitive data into a compliant environment after the fact can create complications.
- Treating Business Critical as just a premium version of Enterprise. If your data falls under HIPAA, PCI, or similar regulations, the additional controls are not optional.
- Dismissing VPS as unnecessary without consulting compliance or legal teams. If your policy requires physical isolation, VPS is the only valid option.
- Focusing only on compute costs. Storage is billed separately at approximately 23 USD per TB per month. Regularly review and clean up unused data.


## Edition Selection Flowchart

```mermaid
flowchart TD
    Q1[Start: What type of data are you processing] 
    Q1 --> Q2[Is the data public or internal only]
    Q2 -->|Yes| Q3[Do you need automatic multi-cluster scaling]
    Q2 -->|No - data is sensitive or regulated| Q4[Do you need HIPAA PCI or FedRAMP compliance]
    
    Q3 -->|No| A[Standard]
    Q3 -->|Yes| B[Enterprise]
    
    Q4 -->|Yes| Q5[Do you require customer managed keys and private networking]
    Q4 -->|No| B
    
    Q5 -->|Yes| Q6[Do you require physical hardware isolation]
    Q5 -->|No| C[Business Critical]
    
    Q6 -->|Yes| D[VPS]
    Q6 -->|No| C
```


## Key Takeaways

- Begin with the simplest edition that meets your current needs. Standard is sufficient for learning and small scale projects.
- Upgrade when your workload or compliance requirements demand it, not in anticipation of future needs.
- Let regulatory requirements drive your edition choice. If a rule mandates isolation or specific controls, that requirement overrides cost considerations.
- VPS is designed for a narrow set of use cases. Unless you operate in banking, healthcare, government, or similar high compliance environments, you likely do not need it.

Think of editions like transportation options:
- Standard gets you where you need to go reliably
- Enterprise adds capacity and flexibility for growing demands
- Business Critical adds protection and controls for sensitive cargo
- VPS provides a private, dedicated route with no external exposure

Choose based on your actual requirements, not on hypothetical future scenarios.
