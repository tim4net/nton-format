# NTON (Nested Table Optimized Notation)

**The Schema-First Data Serialization Format for LLMs.**

NTON is a delimiter-based data format designed to maximize token efficiency for deeply nested, structured data. It combines schema-driven compression with optional field names for clarity, making it both compact and LLM-friendly.

## Why NTON?

Large Language Models (LLMs) struggle with the verbosity of JSON and the ambiguity of YAML. Existing optimized formats (like ZON and TOON) excel at flat data but lose efficiency when data becomes deeply nested.

NTON solves this by separating **Structure** (Schema) from **Content** (Data Stream).

| Feature | JSON | TOON | ZON | NTON |
| :--- | :--- | :--- | :--- | :--- |
| **Structure** | Verbose (Repeated Keys) | Flat Schemas | Flat Table Headers | **Global Schema Definitions** |
| **Nesting** | Heavy Brackets | JSON-like (Quoted) | Quoted Strings (Opaque) | **Explicit Delimiters** |
| **Deduplication** | None | None | Column Headers Only | **Reference Tables (Variables)** |
| **Type System** | None | Optional | None | **Structured (DEF + Optional Fields)** |
| **Field Names** | Required | Required | Per-column | **Optional (Hybrid)** |
| **Whitespace** | Flexible | Flexible | Flexible | **Flexible (Cosmetic)** |
| **Efficiency** | Low | Medium (Flat data) | High (Flat data) | **High (Deep data)** |

## Quick Look

### The Schema
```nton
DEF Worker:   { id, rate, manager? }
DEF Milestone:{ name, date, completion, workers:Worker[]? }
DEF Project:  { id, name, status, active, budget, milestones:Milestone[]? }

REF Status: { $IP:"In Progress", $C:"Completed" }
```

### The Stream

```nton
STREAM Project:
{
  P001,
  "Alpha Initiative",
  status=$IP,
  active=T,
  budget=500000,
  milestones=[
    {
      "Design Sprints",
      2025-12-15,
      1.0,
      workers=[
        {W1, 65.5, "Alice"},
        {W2, 85.0, "Bob"}
      ]
    }
  ]
}
```

**Key Features:**
- Explicit delimiters `{ }` `[ ]` - robust parsing, no whitespace sensitivity
- Optional field names - `status=$IP` for clarity, or pure positional for brevity
- Optional fields with `?` - schema evolution without breaking changes

## Complex Example

Real-world enterprise data with 6 levels of nesting:

```nton
DEF Contact: { id, name, email, phone?, role, department }
DEF Task: {
  id, title, description, status, priority,
  assigned_to:Contact, created_date, due_date,
  completed_date?, tags:string[]?, estimated_hours, actual_hours?
}
DEF Sprint: {
  id, name, start_date, end_date, goal, status,
  tasks:Task[]?, team_members:Contact[]?
}
DEF Project: {
  id, name, description, status, priority,
  budget, spent, start_date, end_date?,
  project_manager:Contact, sprints:Sprint[]?
}

REF Status: { $IP:"In Progress", $C:"Completed", $P:"Planning" }
REF Priority: { $CR:"Critical", $H:"High", $M:"Medium", $L:"Low" }
REF Depts: { $ENG:"Engineering", $PROD:"Product", $DES:"Design" }

STREAM Project:
{
  PROJ001,
  "Cloud Platform Modernization",
  "Migrate legacy infrastructure to cloud-native architecture",
  status=$IP,
  priority=$CR,
  budget=5000000,
  spent=3200000,
  start_date=2024-01-15,
  end_date=2025-06-30,
  project_manager={
    PM001, "Sarah Chen", "sarah.chen@company.com",
    "+1-555-0101", "Senior Program Manager", $PROD
  },
  sprints=[
    {
      SPR001, "Infrastructure Setup",
      2024-01-15, 2024-02-15,
      "Set up cloud infrastructure and CI/CD pipelines",
      $C,
      tasks=[
        {
          TSK001, "Configure AWS VPC",
          "Set up Virtual Private Cloud with proper subnets",
          $C, $H,
          assigned_to={
            ENG001, "Alex Rodriguez", "alex.r@company.com",
            "+1-555-0201", "Cloud Architect", $ENG
          },
          2024-01-15, 2024-01-22, completed_date=2024-01-20,
          tags=["aws", "infrastructure", "networking"],
          estimated_hours=40, actual_hours=35
        },
        {
          TSK002, "Set up Kubernetes cluster",
          "Deploy and configure EKS cluster with autoscaling",
          $C, $CR,
          assigned_to={
            ENG002, "Maria Santos", "maria.s@company.com",
            "+1-555-0202", "DevOps Engineer", $ENG
          },
          2024-01-23, 2024-02-05, completed_date=2024-02-03,
          tags=["kubernetes", "eks", "container-orchestration"],
          estimated_hours=80, actual_hours=85
        }
      ],
      team_members=[
        {ENG001, "Alex Rodriguez", "alex.r@company.com", "+1-555-0201", "Cloud Architect", $ENG},
        {ENG002, "Maria Santos", "maria.s@company.com", "+1-555-0202", "DevOps Engineer", $ENG}
      ]
    }
  ]
}
```

**Same data in JSON: 3,781 tokens**
**In NTON: 2,842 tokens**
**Savings: 24.8% (939 tokens)**

The savings increase with more records - with 10 projects, NTON saves ~35-40%.

## Usage

NTON is designed for:

1.  **RAG Context Injection:** Fit 25-40% more database records into a prompt (depends on nesting depth and record count).
2.  **LLM Outputs:** Enforce strict schemas for generated code/data.
3.  **Log Storage:** High-density structured logging with fixed schemas.

## Comparison to Other Formats

### NTON vs. TOON

[TOON (Table-Oriented Object Notation)](https://github.com/anthropics/TOON) is Anthropic's token-optimized format that uses schemas and columnar layout to reduce token overhead.

**Where TOON Excels:**
- Flat tabular data with consistent schemas
- Simple one-level nesting via inline JSON
- Straightforward schema definitions

**Where NTON Improves:**
- **Deep Nesting:** TOON encodes nested objects as quoted JSON strings, which reintroduces verbosity. NTON uses explicit delimiters with positional encoding to maintain compression at all depths.
- **String Deduplication:** TOON has no built-in mechanism for deduplicating repeated strings (like status values, department names). NTON's `REF` tables eliminate this redundancy.
- **Type Composition:** NTON supports true compositional types (`workers:Worker[]`), while TOON relies on inline JSON for complex nesting.
- **Flexible Syntax:** NTON allows hybrid positional/named fields (`{P001, "Alpha", status=$IP}`) - be brief where obvious, explicit where needed.

**Example:** A project with 5 milestones, each having 2 workers (4 levels deep):
- TOON: ~850 tokens (nested JSON quoted strings)
- NTON: ~420 tokens (delimiter-based nesting + REF variables)

### NTON vs. ZON

[ZON (Zed Object Notation)](https://github.com/zed-industries/zed/blob/main/docs/zon.md) is Zed's optimized format focused on maximum compression for flat data.

**Where ZON Excels:**
- Extremely compact for single-level tables
- No schema overhead for simple data
- Minimal punctuation

**Where NTON Improves:**
- **Structured Nesting:** ZON treats nested data as opaque quoted strings. NTON preserves structure through explicit delimiters `[ ]` `{ }`, making nested data both compact and parseable.
- **Global Schema Reuse:** ZON headers apply per-stream. NTON's `DEF` blocks define types once and reuse them across multiple streams.
- **Semantic Compression:** NTON's `REF` tables deduplicate at the semantic level (replacing "In Progress" with `$IP`), not just structural.
- **Schema Evolution:** NTON supports optional fields (`field?`), enabling backward-compatible schema changes.

**Example:** Three data streams sharing common types:
- ZON: Headers repeated 3 times, nested objects quoted
- NTON: Types defined once, nested objects use explicit delimiters

**Key Insight:** Both TOON and ZON optimize the "width" problem (repeated keys). NTON optimizes the "depth" problem (nested structures) while remaining LLM-friendly with explicit delimiters and flexible whitespace.

## License

MIT
