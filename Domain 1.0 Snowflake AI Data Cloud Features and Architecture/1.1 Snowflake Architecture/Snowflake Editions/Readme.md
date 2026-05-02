# Snowflake Editions Overview

## Edition Hierarchy

```mermaid
graph LR
    A[Standard] --> B[Enterprise]
    B --> C[Business Critical]
    C --> D[VPS]
```

Each edition includes all features from the edition below it plus additional capabilities [[5]].

---

## Feature Comparison Table

| Feature Category | Standard | Enterprise | Business Critical | VPS |
|-----------------|----------|------------|-------------------|-----|
| Core SQL and data operations | Yes | Yes | Yes | Yes |
| Automatic data encryption | Yes | Yes | Yes | Yes |
| Standard Time Travel (1 day) | Yes | Yes | Yes | Yes |
| Extended Time Travel (up to 90 days) | No | Yes | Yes | Yes |
| Multi-cluster warehouses | No | Yes | Yes | Yes |
| Row and column level security | No | Yes | Yes | Yes |
| Customer managed encryption keys | No | No | Yes | Yes |
| Private network connectivity | No | No | Yes | Yes |
| HIPAA and PHI compliance support | No | No | Yes | Yes |
| Automated failover and failback | No | No | Yes | Yes |
| Dedicated isolated hardware | No | No | No | Yes |
| Marketplace access | Yes | Yes | Yes | No |

Sources: [[5]][[2]]

---

## Edition Details

### Standard Edition

```mermaid
quadrantChart
    title "Standard Edition Fit"
    x-axis "Simple" --> "Complex"
    y-axis "Low Risk" --> "High Risk"
    "Learning projects": [0.2, 0.2]
    "Small teams": [0.3, 0.3]
    "Proof of concept": [0.25, 0.25]
```

- Entry level access to core Snowflake functionality [[5]]
- Full SQL support with standard data types
- Automatic encryption for all data
- 1 day Time Travel for data recovery
- Best for teams evaluating Snowflake or running simple workloads

### Enterprise Edition

```mermaid
quadrantChart
    title "Enterprise Edition Fit"
    x-axis "Simple" --> "Complex"
    y-axis "Low Risk" --> "High Risk"
    "Growing data teams": [0.6, 0.4]
    "Production analytics": [0.7, 0.5]
    "Multi-team environments": [0.65, 0.45]
```

- Includes all Standard features plus [[5]]
- Extended Time Travel up to 90 days
- Multi-cluster warehouses for handling concurrency
- Row and column level security controls
- Materialized views and query acceleration
- Best for organizations with multiple teams or variable workloads

### Business Critical Edition

```mermaid
quadrantChart
    title "Business Critical Edition Fit"
    x-axis "Simple" --> "Complex"
    y-axis "Low Risk" --> "High Risk"
    "Healthcare PHI data": [0.8, 0.9]
    "Financial services": [0.85, 0.85]
    "Government compliance": [0.9, 0.95]
```

- Includes all Enterprise features plus [[5]]
- Customer managed encryption keys via Tri-Secret Secure
- Private connectivity options for network isolation
- Support for HIPAA, PCI DSS, FedRAMP compliance
- Automated failover and failback for disaster recovery
- Best for regulated industries handling sensitive data

### Virtual Private Snowflake VPS

```mermaid
quadrantChart
    title "VPS Edition Fit"
    x-axis "Shared" --> "Isolated"
    y-axis "Standard" --> "Maximum Security"
    "Financial institutions": [0.95, 0.98]
    "Defense workloads": [0.98, 0.99]
    "Ultra sensitive data": [0.97, 0.97]
```

- Includes all Business Critical features [[5]]
- Runs in a completely separate Snowflake environment
- No hardware or resource sharing with other customers
- Dedicated metadata store and compute pool
- Best for organizations requiring maximum isolation


## Pricing Overview (US East AWS On Demand)

| Edition | Price Per Credit | Storage Cost |
|---------|-----------------|--------------|
| Standard | 2.00 USD | 23.00 USD per TB per month |
| Enterprise | 3.00 USD | 23.00 USD per TB per month |
| Business Critical | 4.00 USD | 23.00 USD per TB per month |
| VPS | Contact sales | Contact sales |

Source: [[2]]

Note: Credits are consumed by compute operations. You pay only for what you use [[6]].


## Security Feature Progression

```mermaid
flowchart TD
    S[Standard] -->|adds| E[Enterprise]
    E -->|adds| BC[Business Critical]
    BC -->|adds| V[VPS]
    
    subgraph "Added in Enterprise"
        E1[Extended Time Travel 90 days]
        E2[Row and column masking]
        E3[Multi-cluster scaling]
        E4[Materialized views]
    end
    
    subgraph "Added in Business Critical"
        BC1[Customer managed keys]
        BC2[Private network links]
        BC3[HIPAA PHI support]
        BC4[Automated failover]
    end
    
    subgraph "Added in VPS"
        V1[Dedicated hardware]
        V2[No resource sharing]
        V3[Isolated environment]
    end
```


## Time Travel and Data Recovery

| Edition | Time Travel Window | Fail-safe |
|---------|-------------------|-----------|
| Standard | 1 day | 7 days read only |
| Enterprise | Up to 90 days configurable | 7 days read only |
| Business Critical | Up to 90 days configurable | 7 days read only |
| VPS | Up to 90 days configurable | 7 days read only |

Source: [[23]][[26]]

Note: Fail-safe is automatic and cannot be disabled. It provides emergency recovery only [[26]].


## Selection Decision Flow

```mermaid
flowchart TD
    Q1[Start: What data are you processing] 
    Q1 --> Q2[Is data public or internal only]
    Q2 -->|Yes| Q3[Need automatic scaling for concurrency]
    Q2 -->|No - sensitive or regulated| Q4[Need HIPAA PCI or FedRAMP]
    
    Q3 -->|No| A[Standard]
    Q3 -->|Yes| B[Enterprise]
    
    Q4 -->|Yes| Q5[Need customer managed keys and private networking]
    Q4 -->|No| B
    
    Q5 -->|Yes| Q6[Need physical hardware isolation]
    Q5 -->|No| C[Business Critical]
    
    Q6 -->|Yes| D[VPS]
    Q6 -->|No| C
```


## Common Selection Mistakes

- Choosing Standard for production workloads that need scaling. Multi-cluster warehouses require Enterprise or higher [[5]].
- Assuming Business Critical is optional for regulated data. If HIPAA or PCI applies, the additional controls are required not optional [[5]].
- Overlooking storage costs. Compute credits get attention but storage is billed separately at 23 USD per TB per month [[2]].
- Selecting VPS without confirming isolation requirements. VPS is for physical separation. If policy does not require it, Business Critical is usually sufficient [[5]].


## Key Points

- Start with the edition that matches your current requirements. You can change editions later as needs evolve [[5]].
- Compliance requirements should drive edition choice. If regulations mandate specific controls, select the edition that provides them [[5]].
- All editions use the same query engine. Performance is not reduced on lower tiers [[5]].
- VPS is designed for a narrow set of use cases. Unless you require physical isolation, you likely do not need it [[5]].

Think of editions as layers of capability:
- Standard provides the foundation
- Enterprise adds scale and governance
- Business Critical adds compliance and protection
- VPS adds physical isolation

Choose based on your actual data, risk, and regulatory requirements.
