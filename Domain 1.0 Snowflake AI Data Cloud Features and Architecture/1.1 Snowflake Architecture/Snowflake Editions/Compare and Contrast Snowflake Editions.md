# Compare and Contrast Snowflake Editions

# Snowflake Editions: Simple Comparison

## The Big Picture

```mermaid
graph LR
    A[Standard] --> B[Enterprise]
    B --> C[Business Critical]
    C --> D[VPS]
    
    style A fill:#e1f5fe
    style B fill:#b3e5fc
    style C fill:#81d4fa
    style D fill:#4fc3f7
```

Each level = everything below it + extra stuff.

---

## Quick Feature Table

| What you get | Standard | Enterprise | Business Critical | VPS |
|-------------|----------|------------|-------------------|-----|
| Core SQL + data loading | ✅ | ✅ | ✅ | ✅ |
| Basic security + encryption | ✅ | ✅ | ✅ | ✅ |
| Time Travel (undo mistakes) | 1 day | 90 days | 90 days | 90 days |
| Multi-cluster warehouses | ❌ | ✅ | ✅ | ✅ |
| Row/column level security | ❌ | ✅ | ✅ | ✅ |
| Customer-managed keys | ❌ | ❌ | ✅ | ✅ |
| Private network connections | ❌ | ❌ | ✅ | ✅ |
| HIPAA/PHI support | ❌ | ❌ | ✅ | ✅ |
| Fully isolated hardware | ❌ | ❌ | ❌ | ✅ |
| Disaster recovery failover | ❌ | ❌ | ✅ | ✅ |

---

## Who Should Pick What?

### Standard
```mermaid
quadrantChart
    title "Standard Edition Sweet Spot"
    x-axis "Low Complexity" --> "High Complexity"
    y-axis "Low Risk" --> "High Risk"
    "Startups": [0.2, 0.2]
    "Small teams": [0.3, 0.3]
    "Dev/Test": [0.25, 0.25]
```

- You're testing Snowflake
- Small team, simple data
- No strict compliance rules
- Budget matters most

### Enterprise
```mermaid
quadrantChart
    title "Enterprise Edition Sweet Spot"
    x-axis "Low Complexity" --> "High Complexity"
    y-axis "Low Risk" --> "High Risk"
    "Growing companies": [0.6, 0.4]
    "Data teams": [0.7, 0.5]
    "Analytics at scale": [0.65, 0.45]
```

- You need more power + control
- Multiple teams running queries
- Want to keep deleted data longer (90 days)
- Need fine-grained access rules

### Business Critical
```mermaid
quadrantChart
    title "Business Critical Sweet Spot"
    x-axis "Low Complexity" --> "High Complexity"
    y-axis "Low Risk" --> "High Risk"
    "Healthcare": [0.8, 0.9]
    "Finance": [0.85, 0.85]
    "Gov/Regulated": [0.9, 0.95]
```

- You handle sensitive data (health, finance, gov)
- Must meet HIPAA, PCI, FedRAMP
- Need your own encryption keys
- Can't afford downtime → need failover

### VPS (Virtual Private Snowflake)
```mermaid
quadrantChart
    title "VPS Sweet Spot"
    x-axis "Shared Resources" --> "Fully Isolated"
    y-axis "Standard Security" --> "Maximum Isolation"
    "Banks": [0.95, 0.98]
    "Defense": [0.98, 0.99]
    "Ultra-sensitive data": [0.97, 0.97]
```

- You need physical separation from other customers
- Highest security + compliance bar
- Budget is not the main constraint
- You're a large institution with strict rules

---

## Cost per Credit (US East, AWS) [[19]]

| Edition | Price/Credit | Best for |
|---------|-------------|----------|
| Standard | ~$2.00 | Learning, small projects |
| Enterprise | ~$3.00 | Most companies [[21]] |
| Business Critical | ~$4.00 | Regulated data |
| VPS | Custom pricing | Talk to sales |

> 💡 Credits = compute power. You pay for what you use, when you use it [[3]].

---

## What Actually Changes Between Editions?

### 🔐 Security & Control
```mermaid
flowchart TD
    S[Standard] -->|add| E[Enterprise]
    E -->|add| BC[Business Critical]
    BC -->|add| V[VPS]
    
    subgraph "Add at Enterprise"
        E1[90-day Time Travel]
        E2[Row/Column masking]
        E3[Multi-cluster warehouses]
    end
    
    subgraph "Add at Business Critical"
        BC1[Your own encryption keys]
        BC2[Private network links]
        BC3[HIPAA/PHI support]
        BC4[Auto failover]
    end
    
    subgraph "Add at VPS"
        V1[Dedicated hardware]
        V2[No shared resources]
        V3[Maximum isolation]
    end
```

### ⚡ Performance & Scale
- **Standard**: One warehouse at a time. Good for simple workloads.
- **Enterprise+**: Spin up multiple warehouses automatically when traffic spikes.
- All editions get the same query engine — no "slow mode" on lower tiers.

### 🔄 Data Recovery
| Edition | How far back can you "undo"? |
|---------|-----------------------------|
| Standard | 24 hours |
| Enterprise+ | Up to 90 days |
| All editions | +7 days in Fail-safe (read-only, emergency only) |

---

## Common Mistakes to Avoid

- ❌ "We'll start with Standard and upgrade later" → Upgrading is easy, but migrating sensitive data later can be messy. Plan ahead.
- ❌ "Business Critical is just Enterprise with a fancy name" → No. If you handle PHI or financial data, you *need* the extra controls. It's not optional.
- ❌ "VPS is overkill" → Maybe. But if your legal/compliance team says "isolated infrastructure", VPS is the only answer.
- ❌ Ignoring storage costs → Compute credits get attention, but storage is $23/TB/month [[19]]. Clean up old data.

---

## Decision Flowchart

```mermaid
flowchart TD
    Q1[Start: What data are you handling?] 
    Q1 --> Q2[Public or internal only?]
    Q2 -->|Yes| Q3[Need multi-cluster scaling?]
    Q2 -->|No - sensitive/regulated| Q4[HIPAA/PCI/FedRAMP?]
    
    Q3 -->|No| A[Standard]
    Q3 -->|Yes| B[Enterprise]
    
    Q4 -->|Yes| Q5[Need customer-managed keys + private networking?]
    Q4 -->|No| B
    
    Q5 -->|Yes| Q6[Need physical hardware isolation?]
    Q5 -->|No| C[Business Critical]
    
    Q6 -->|Yes| D[VPS]
    Q6 -->|No| C
```

---

## Bottom Line

- **Start simple**. Standard is fine for learning and small projects.
- **Upgrade when you hit a wall** — not before. Enterprise unlocks real team workflows.
- **Compliance drives edition**, not features. If the law says "isolate this", you don't get to pick cheap.
- **VPS is rare**. If you're not a bank, hospital, or government, you probably don't need it.

> Think of it like a car:  
> Standard = reliable sedan  
> Enterprise = SUV with extra seats  
> Business Critical = armored vehicle  
> VPS = private convoy on a closed road  

Pick the one that matches your actual risk — not your fear of missing out.
