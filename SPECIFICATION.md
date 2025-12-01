# NTON Specification v0.02

## 1. Introduction
Nested Table Optimized Notation (NTON) is a schema-driven serialization format. It achieves high data density by defining data structures (`DEF`) and recurring values (`REF`) in a header, allowing the data body to use positional encoding with optional field names for clarity.

NTON is designed for LLM contexts where token efficiency matters, while maintaining robustness and evolvability.

## 2. Document Structure
An NTON document consists of three optional sections, which MUST appear in this order:
1.  **Definitions** (`DEF`)
2.  **References** (`REF`)
3.  **Data Stream** (`STREAM`)

### 2.1 Encoding
NTON files MUST be encoded in UTF-8.

## 3. The Header

### 3.1 Type Definitions (DEF)

Types function as structs. They define the fields and their order in the data stream.

**Syntax:** `DEF <TypeName>: { <field>, <field>:<Type>, ... }`

* Fields without a type default to Primitive (string, number, boolean, null).
* Arrays are denoted by `[]`.
* Optional fields are denoted by `?`.

**Examples:**

```nton
# Simple type
DEF User: { id, name, email }

# Optional fields
DEF User: { id, name, email?, phone? }

# Typed fields
DEF Group: { name, owner:User, members:User[] }

# Optional arrays
DEF Project: {
  id,
  name,
  status,
  active,
  manager_id?,
  budget,
  milestones:Milestone[]?
}
```

### 3.2 Reference Tables (REF)

Reference tables allow string deduplication via variable substitution.

**Syntax:** `REF <RefName>: { $<Var>: "<Value>", ... }`

* Variables MUST start with `$`.
* Variables can be used anywhere a string value is expected.

**Example:**

```nton
REF Status: {
  $IP: "In Progress",
  $C: "Completed",
  $P: "Planning"
}

REF Departments: {
  $T: "Technology",
  $M: "Marketing",
  $F: "Finance"
}
```

## 4. The Data Stream

### 4.1 Primitives

* **Booleans:** `T` or `F` (also `true` or `false`)
* **Null:** `null`, `~`, or `_`
* **Strings:** Unquoted if alphanumeric with no spaces. Double-quoted otherwise.
* **Numbers:** Standard integer or floating-point: `42`, `3.14`, `-17`, `1.23e-4`
* **Dates:** ISO 8601 format: `2025-12-15` or `"2025-12-15T10:30:00Z"`
* **Variables:** References to `REF` tables (e.g., `$IP`, `$Alice`)

### 4.2 Objects and Arrays

#### Objects

Objects are enclosed in `{ }` and contain comma-separated values.

**Pure Positional (Most Compact):**
```nton
{U1, "Alice", "alice@example.com"}
```

**Named Fields (Most Readable):**
```nton
{id=U1, name="Alice", email="alice@example.com"}
```

**Hybrid (Best Balance):**
```nton
{U1, "Alice", email="alice@example.com"}  # Mix positional and named
```

#### Arrays

Arrays are enclosed in `[ ]` and contain comma-separated items.

```nton
[
  {U1, "Alice", "alice@example.com"},
  {U2, "Bob", "bob@example.com"}
]
```

#### Nested Structures

Objects and arrays can be nested to any depth using explicit delimiters.

```nton
{
  id=P001,
  name="Alpha Initiative",
  milestones=[
    {
      name="Design Phase",
      workers=[
        {W1, 65.50, "Alice"},
        {W2, 85.00, "Bob"}
      ]
    }
  ]
}
```

### 4.3 Whitespace Rules

**Whitespace is FLEXIBLE:**

* Indentation is **cosmetic only** (for human readability)
* Parser ignores leading/trailing whitespace
* Line breaks are optional
* Any consistent indentation style is acceptable (2 spaces, 4 spaces, tabs)

**All of these are equivalent:**

```nton
{U1, "Alice", "alice@example.com"}

{ U1, "Alice", "alice@example.com" }

{
  U1,
  "Alice",
  "alice@example.com"
}
```

### 4.4 Streams

A stream begins with `STREAM <TypeName>:` followed by one or more records.

**Simple Stream:**

```nton
STREAM User:
{U1, "Alice", "alice@example.com"}
{U2, "Bob", "bob@example.com"}
{U3, "Carol", email=null}  # Optional field omitted
```

**Nested Stream:**

```nton
STREAM Project:
{
  P001,
  "Alpha Initiative",
  status=$IP,
  active=T,
  budget=500000,
  milestones=[
    {name="Design", date=2025-12-15, completion=1.0},
    {name="Development", date=2026-01-20, completion=0.5}
  ]
}
```

### 4.5 Comments

```nton
# Line comment

{U1, "Alice"}  # Inline comment

/*
   Block comment
   spanning multiple lines
*/
```

## 5. Grammar (ABNF Summary)

```abnf
document      = *definition *reference *stream
definition    = "DEF" SP type-name ":" SP "{" field-list "}" LF
field         = field-name [":" type-name] ["?"]
reference     = "REF" SP ref-name ":" SP "{" var-list "}" LF
stream        = "STREAM" SP type-name ":" LF *record
record        = object LF
object        = "{" [field-value *("," field-value)] "}"
field-value   = [field-name "="] value
array         = "[" [value *("," value)] "]"
value         = primitive / object / array / variable
primitive     = string / number / boolean / null / date
```

## 6. Optional Fields and Schema Evolution

### Optional Fields

Fields marked with `?` in the DEF can be omitted or set to `null`.

```nton
DEF User: { id, name, email?, phone? }

# All valid:
{U1, "Alice", "alice@example.com", "(555) 1234"}
{U2, "Bob", "bob@example.com"}
{U3, "Carol", email=null, phone=null}
{U4, "Dave"}  # email and phone omitted
```

### Schema Evolution

Adding optional fields is backward compatible:

```nton
# Version 1
DEF User: { id, name, email }

# Version 2 (backward compatible)
DEF User: { id, name, email, phone?, created_at? }
```

Old data remains valid under the new schema.

## 7. Error Handling

Parsers SHOULD be forgiving:

* **Trailing commas:** Allowed and ignored
* **Missing optional fields:** Treated as `null`
* **Extra unknown fields:** Ignored with a warning
* **Type mismatches:** Attempt coercion, error if impossible

## 8. File Extension and MIME Type

* **Extension:** `.nton`
* **MIME type:** `application/nton`

## 9. Complete Example

```nton
# GlobalTech Project Data - NTON v0.02

DEF Worker: { id, rate, manager_name? }

DEF Milestone: {
  name,
  date,
  completion,
  workers:Worker[]?
}

DEF Project: {
  id,
  name,
  status,
  active,
  manager_id?,
  budget,
  milestones:Milestone[]?
}

REF Status: {
  $IP: "In Progress",
  $C: "Completed",
  $P: "Planning"
}

REF Managers: {
  $Alice: "Alice",
  $Bob: "Bob",
  $Carol: "Carol"
}

STREAM Project:
{
  P001,
  "Alpha Initiative",
  status=$IP,
  active=T,
  manager_id=M1,
  budget=500000,
  milestones=[
    {
      "Design Sprints",
      2025-12-15,
      1.0,
      workers=[
        {W1, 65.50, $Alice},
        {W2, 85.00, $Bob}
      ]
    },
    {
      "Prototype Approval",
      2026-01-20,
      0.9,
      workers=[
        {W1, 65.50, $Alice},
        {W3, 72.25, $Carol}
      ]
    }
  ]
}

{
  P002,
  "HR Portal V2",
  status=$C,
  active=F,
  budget=120000,
  milestones=[
    {"Requirements", 2025-10-01, 1.0},
    {"Launch", 2025-11-20, 1.0}
  ]
}
```

## 10. Key Improvements in v0.02

| Feature | Benefit |
|---------|---------|
| Explicit delimiters `{ }` `[ ]` | Robust parsing, no indentation errors |
| Optional field names | Clarity when needed, brevity when obvious |
| Optional fields `?` | Schema evolution without breaking changes |
| Flexible whitespace | LLM-friendly, human-friendly |
| No count markers | Simpler, less error-prone |
| Hybrid syntax | Balance between compression and readability |
