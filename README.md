# Error Message Specification

**Version:** 1.0.0  
**Purpose:** This document defines the structure, semantics, and construction rules for error messages.

It is designed to be **mechanically precise** and **strictly actionable**.  
Each error message communicates five things unambiguously:

1. **The violation** (Code)
2. **The entity** (Entity)
3. **The location** (Path)
4. **The issue** (Issue)
5. **The fix** (Action)

Errors are generated through a **[five-step construction process](#step-1-identify-the-failure-constraints)**.

> **Reference implementation:** [`sass-error`](https://github.com/nicholasgillespie/sass-error) — Sass utility functions for building error messages.

---

## The Structure of an Error

Every error message is a single structured string composed of a machine-readable code and a human-readable message.

```scss
[CODE] <EntityType> "<EntityName>" [@ <Path>]: <Issue> → <Action> [context: ...]
```

### Fields

- **CODE** - Error code.
- **EntityType** - Type of the validated entity.
- **EntityName** - Name of the entity.
- **Path** - Location of the failure within the entity.
- **Issue** - Description of the failure.
- **Action** - Recommended corrective action.
- **Context** - Additional information relevant to the failure.

### Example in Practice

```scss
[TOKEN_TIER_VALUE] Token "font-weight" @ settings > tier: Invalid value "random" → Allowed: primitive | semantic | component
```

> **Note:** See the **[Annex: Anatomy of a Failure](#annex-anatomy-of-a-failure)** section for a step-by-step breakdown of how this specific message is constructed.

---

## Step 1: Identify the Failure (Constraints)

When validation fails, classify the violation using the constraints below. Evaluate these in priority order and stop at the first detected failure.

| Priority | Category       | Constraint   | Description                                                                                       |
| :------: | -------------- | ------------ | ------------------------------------------------------------------------------------------------- |
|    1     | **Presence**   | `MISSING`    | Required attribute is absent.                                                                     |
|    1     |                | `UNKNOWN`    | Attribute is not recognized by the schema.                                                        |
|    2     | **Permission** | `FORBIDDEN`  | Syntax is valid but disallowed in this specific location.                                         |
|    3     | **Integrity**  | `TYPE`       | Value has an incorrect data type (`null`, `bool`, `number`, `string`, `color`, `list`, `map`).    |
|    3     |                | `EMPTY`      | Value has the correct type but contains no content (`null`, `""`, empty list/map `()`).           |
|    3     |                | `INVALID`    | Covers both `TYPE` and `EMPTY` with a single convenience check.                                   |
|    4     | **Value**      | `SHAPE`      | Structure of the value is invalid or incomplete (e.g., wrong nesting, wrong count, missing unit). |
|    4     |                | `VALUE`      | Value has the correct type and shape but fails an allowed-values check (e.g., whitelist, range).  |
|    5     | **State**      | `CONFLICT`   | Two valid attributes/values cannot be used together.                                              |
|    6     | **Reference**  | `UNRESOLVED` | Target resource referenced by a valid value cannot be found.                                      |

> **Note:** `INVALID` is a convenience shortcut combining `TYPE` and `EMPTY`, applicable when both failures share the same fix. Use `TYPE` and `EMPTY` individually when distinct guidance is required.

---

## Step 2: Generate the Error Code

Construct an error code to prepend to the message. Assemble the code using the following segments:

```scss
ENTITY[_ATTRIBUTE][_ID]_CONSTRAINT
```

### Segments

| Segment       | Presence    | Description                                                                      |
| ------------- | ----------- | -------------------------------------------------------------------------------- |
| `ENTITY`      | Required    | Type of the validated entity (e.g., `VARIABLE`, `ARGUMENT`, `TOKEN`, `SCHEMA`).  |
| `_ATTRIBUTE`  | Conditional | Named member or identifier within the entity.                                    |
| `_ID`         | Conditional | Indicator denoting that the error concerns the identifier of a collection entry. |
| `_CONSTRAINT` | Required    | Constraint selected in [Step 1](#step-1-identify-the-failure-constraints).       |

### Composition Rules

| Scope                     | Pattern                                                               | Example                      |
| ------------------------- | --------------------------------------------------------------------- | ---------------------------- |
| **Whole Entity**          | Use only the entity type and constraint.                              | `TOKEN_TYPE`                 |
| **Named Member**          | Append the attribute name.                                            | `TOKEN_VALUES_TYPE`          |
| **Collection Value**      | Append the attribute name and the semantic `ENTRY` label.             | `TOKEN_VALUES_ENTRY_TYPE`    |
| **Collection Identifier** | Use the parent attribute, the `ENTRY` label, and the `_ID` indicator. | `TOKEN_VALUES_ENTRY_ID_TYPE` |

> **Note:** `ENTRY` may be replaced with a semantic label (e.g., `FIELD`) when it better represents the entry's domain role. Any such label is treated as equivalent to `ENTRY` or `ENTRY_ID`.

### Entity Identification

Select the `EntityType` and `EntityName` that match the entity under validation.

| EntityType | Description                    | Example                       |
| ---------- | ------------------------------ | ----------------------------- |
| `Variable` | Global variable                | `Variable "$TOKENS_REGISTRY"` |
| `Argument` | Function parameter             | `Argument "$token-key"`       |
| `Token`    | Token registry entry           | `Token "font-weight"`         |
| `Schema`   | Schema definition or validator | `Schema "Token"`              |

> **Note:** When an argument carries a specific token's data, use the token as the entity — `Token "<token-key>"` over `Argument "$token-key"`.

---

## Step 3: Pinpoint the Location (Path)

If the error occurs within a nested structure, provide a hierarchical breadcrumb trail to the exact location.

```scss
@ settings > tier
```

| Location Type             | Path Construction Rule                                                  | Example                     |
| ------------------------- | ----------------------------------------------------------------------- | --------------------------- |
| **Root Presence**         | Omit the path entirely.                                                 | _(empty)_                   |
| **Nested Presence**       | Point to the parent attribute of the missing/unknown attribute.         | `@ settings`                |
| **Collection Identifier** | Point to the entry identifier, replacing the final segment with `<id>`. | `@ values > core > <id>`    |
| **Attribute / Entry**     | Point directly to the attribute/entry containing the error.             | `@ values > core > primary` |

> **Note:** `<id>` is a literal placeholder — used when the key itself fails a `TYPE` or `EMPTY` constraint and cannot be rendered.

---

## Step 4: Draft the Message (Templates)

To maintain predictability, generate human-readable strings strictly from these templates.

| Constraint   | Issue Template                    | Action Template                  |
| ------------ | --------------------------------- | -------------------------------- |
| `MISSING`    | `Missing <kind> "<attribute>"`    | `Include "<attribute>"`          |
| `UNKNOWN`    | `Unknown <kind> "<attribute>"`    | `Remove "<attribute>"`           |
| `FORBIDDEN`  | `Forbidden <kind> "<received>"`   | `Remove "<received>"`            |
| `TYPE`       | `Incorrect type "<received>"`     | `Expected: <expected>`           |
| `EMPTY`      | `Empty value`                     | `Expected: non-empty <expected>` |
| `INVALID`    | `Invalid value`                   | `Expected: non-empty <expected>` |
| `SHAPE`      | `Invalid shape "<received>"`      | `Expected: <expected>`           |
| `VALUE`      | `Invalid <kind> "<received>"`     | `Allowed: <expected>`            |
| `CONFLICT`   | `Conflicting <kind> "<received>"` | `Expected: <expected>`           |
| `UNRESOLVED` | `Unresolved <kind> "<received>"`  | `Reference <expected>`           |

> **Note**: Actions are imperative (`Include`, `Remove`, etc.) when the fix is direct; declarative (`Expected`, `Allowed`) when it requires structural judgment.

### Template Extensions

Templates may be extended with specific patterns for complex scenarios:

| Scenario                    | Target | Pattern                        | Example                                                      |
| --------------------------- | ------ | ------------------------------ | ------------------------------------------------------------ |
| Dependency qualifier        | Issue  | `required for <kind> "<name>"` | `Missing attribute "width" required for feature "clamping"`  |
| Scope qualifier             | Issue  | `in <scope>`                   | `Incorrect type "number" in list`                            |
| Comparison target           | Issue  | `and "<received>"`             | `Conflicting units "px" and "em"`                            |
| Value type hint             | Action | `with value type: <expected>`  | `Include "tier" with value type: string`                     |
| Parameter mapping           | Action | `as <parameter>`               | `Include "mode" as $function-argument`                       |
| Multiple corrective actions | Action | `or <alternative>`             | `Include "width" or register a viewport token under "<key>"` |

> **Note:** Extensions must preserve the action type of their base template (imperative vs declarative) and must not introduce ambiguity in resolution.

### Placeholder Definitions

- `<kind>` - Descriptive category label (e.g., `attribute`, `function`, `feature`, `setting`, `value`).
- `<attribute>` - Attribute name being validated.
- `<received>` - Received value, type, or identifier.
- `<expected>` - Expected type, allowed values, structural requirement, or valid resource.
- `<name>` - Name of the external named entity referenced in a dependency qualifier (e.g., `"clamping"`, `"select-theme"`).
- `<scope>` - Structural container providing positional context (e.g., `list`).
- `<parameter>` - Sass function parameter name (e.g., `$function-argument`).
- `<alternative>` - Full text of the alternative corrective action.
- `<key>` - Registry entry identifier (e.g., a token key or named registry constant).

---

## Step 5: Add Context

If the caller is a named entry point, or if the error code alone doesn't identify the call site, scope, or active mode, append a context block.

```scss
[context: <origin-type> "<origin-name>", token "<token-key>", mode "<mode>"]
```

| Environmental Context | Inclusion Rule                                              | Example                                 |
| --------------------- | ----------------------------------------------------------- | --------------------------------------- |
| **Named entry point** | Function or mixin is a named entry point.                   | `[context: mixin "registry-variables"]` |
| **Cross-boundary**    | Validation originates from a different function or scope.   | `[context: function "compile-token"]`   |
| **Iteration**         | Validation executes within a loop requiring disambiguation. | `[context: token "font-weight"]`        |
| **Mode-dependent**    | Mode-specific restriction causes the failure.               | `[context: mode "strict"]`              |

> **Tip:** Include context when the source of the failure is unclear.

---

## Annex: Anatomy of a Failure

To demonstrate the construction rules in action, consider a `font-weight` token being validated.

### The Violation

In this scenario, the `tier` attribute within the `settings` map contains an invalid value.

```scss
// $TOKENS_REGISTRY key: 'font-weight'
$token: (
  'settings': (
    'tier': 'random',
    // Error: fails whitelist requirement
    'css-properties': (
        'font-weight',
      ),
  ),
  'values': (
    // ...
  ),
);
```

### The Construction Process

When the failure is detected, the [five-step construction process](#step-1-identify-the-failure-constraints) is applied:

| Step                 | Action                                                | Result                                                                 |
| -------------------- | ----------------------------------------------------- | ---------------------------------------------------------------------- |
| **1. Constraint**    | Classify the violation using Priority 4 rules.        | `VALUE`                                                                |
| **2. Code & Entity** | Identify the entity and apply the Named Member rule.  | `[TOKEN_TIER_VALUE]` for `Token "font-weight"`                         |
| **3. Path**          | Pinpoint the location using the Nested Presence rule. | `@ settings > tier`                                                    |
| **4. Templates**     | Draft the message using `VALUE` templates.            | `Invalid value "random" → Allowed: primitive \| semantic \| component` |
| **5. Context**       | Omit the block; entity provides sufficient scope.     | _(omitted)_                                                            |

### Final Result

The five steps are combined to form the final structured string:

```scss
[TOKEN_TIER_VALUE] Token "font-weight" @ settings > tier: Invalid value "random" → Allowed: primitive | semantic | component
```

> **Read as**: Token "font-weight", at settings > tier, has an invalid tier value "random" — set it to one of: primitive, semantic, or component.

[Back to top](#error-message-specification)
