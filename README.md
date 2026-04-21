# Transformers Guide

This document explains all transformers currently supported by Lightstream, how to configure them in YAML, and what each one does to the payload.

Transformers run in order for each target.

## Common structure

Each transformer is declared under `targets[].transformers[]` with a `type` field:

```yaml
targets:
  - name: my_target
    transformers:
      - type: PickFieldsTransformer
        fields_to_pick: [id, name]
```

---

## 1) `EntityRenameTransformer`

Renames the record entity (and optionally namespace).

- Config key: `entity_rename_mapping`
- Supports mapping by:
  - full name: `namespace.entity`
  - entity-only: `entity`

### YAML example

```yaml
- type: EntityRenameTransformer
  entity_rename_mapping:
    public.users: crm.customers
    orders: sales_orders
```

---

## 2) `OmitFieldsTransformer`

Removes fields from `NewPayload` and `OldPayload`.

- Config key: `fields_to_omit`

### YAML example

```yaml
- type: OmitFieldsTransformer
  fields_to_omit:
    - password_hash
    - internal_notes
```

---

## 3) `PickFieldsTransformer`

Keeps only selected fields in `NewPayload`.

- Config key: `fields_to_pick`
- Note: `OldPayload` is not filtered by this transformer.

### YAML example

```yaml
- type: PickFieldsTransformer
  fields_to_pick:
    - id
    - name
    - created_at
```

---

## 4) `RenameFieldsTransformer`

Renames field keys in both `NewPayload` and `OldPayload`.

- Config key: `rename_fields_mapping`

### YAML example

```yaml
- type: RenameFieldsTransformer
  rename_fields_mapping:
    name: user_name
    email: user_email
```

---

## 5) `FieldMappingTransformer`

Maps field names like `RenameFieldsTransformer` (current runtime behavior is equivalent).

- Config key: `field_mapping`
- Applies to both `NewPayload` and `OldPayload`.

### YAML example

```yaml
- type: FieldMappingTransformer
  field_mapping:
    status: subscription_status
```

---

## 6) `SleepTransformer`

Pauses processing for a fixed amount of milliseconds.

- Config key: `sleep_time` (milliseconds)

### YAML example

```yaml
- type: SleepTransformer
  sleep_time: 250
```

---

## 7) `DatabaseLookupTransformer`

Enriches the payload by running a SQL lookup and merging returned columns into both payloads.

- Config keys:
  - `database_type` (currently `postgres`)
  - `connection_string`
  - `table`
  - `where`
  - `fields`

### YAML example

```yaml
- type: DatabaseLookupTransformer
  database_type: postgres
  connection_string: ${BILLING_DB_CONNECTION_STRING}
  table: subscriptions
  where: "user_id = {{id}}"
  fields:
    - plan_name
    - status AS subscription_status
```

### Notes

- `where` supports handlebars placeholders like `{{id}}`.
- Placeholders are converted to SQL parameters internally.
- If lookup fails at runtime, current behavior is to continue without enrichment.

---

## 8) `ValueMapperTransformer`

Sets a field value using lookup, rules, or default fallback.

- Config keys:
  - `field` (field to read/write)
  - `lookup` (exact value map)
  - `rules` (ordered rules)
  - `default` (used when no lookup/rule matches)

### YAML example (lookup)

```yaml
- type: ValueMapperTransformer
  field: role
  lookup:
    1: ADMIN
    2: MANAGER
  default: USER
```

### YAML example (rule)

```yaml
- type: ValueMapperTransformer
  field: access_level
  rules:
    - type: match
      match: "{{roleName}} == 'ADMIN'"
      value: FULL
    - type: match
      match: "EDITOR"
      value: WRITE
  default: READ
```

### Rule behavior

- Supported rule type: `match`
- Handles:
  - direct matches (`"EDITOR"`)
  - handlebars template values (`"{{roleName}}"`)
  - simple expression-style equality (`"{{roleName}} == 'ADMIN'"`)

---

## Validation summary

Your config validation currently checks:

- required keys per transformer
- non-empty lists/maps where needed
- `database_type` limited to `postgres` for lookup transformer
- value mapper rule shape (`type`, `match`, `value`)

If a transformer block has wrong keys (example: `fields_to_pick` under `OmitFieldsTransformer`), pipeline build can fail, so prefer running config validation before startup logic.
