# YAML Configuration Guide

This guide documents the full Lightstream YAML shape: root config, sources, targets, loaders, and all transformers.

## Loader behavior

The YAML loader reads all `.yaml` and `.yml` files inside the `lightstream` folder, expands environment variables, and merges the result.

- `store_connection_string` and `store_database_name` must not conflict across files.
- `sources` and `targets` are appended from every file.
- Environment variables are expanded with `${VAR_NAME}`.

## Root config structure

```yaml
store_connection_string: ${STORE_CONNECTION_STRING}
store_database_name: ${STORE_DATABASE_NAME}

sources:
  - name: user_db
    type: postgres
    mode: notify
    connection_string: ${USER_DB_CONNECTION_STRING}
    tables: [users]

targets:
  - name: webhook
    pipe_name: user_events
    type: http
    source_name: user_db
    http_details:
      endpoint: ${WEBHOOK_URL}
      method: POST
      timeout_ms: 10000
      max_concurrent_requests: 8
      headers:
        Content-Type: application/json
      body_template: |
        {
          "id": {{id}}
        }
```

## Sources

Source schema:

```yaml
sources:
  - name: <string>
    type: postgres
    mode: notify
    connection_string: <string>
    tables:
      - <schema_or_table_ref>
```

Notes:

- `type` currently supports only `postgres`.
- `mode` currently supports only `notify`.
- `tables` are required for trigger attachment flow.

## Targets

A target must always have:

- `name`
- `pipe_name`
- `type` (`postgres`, `mongo`, or `http`)
- `source_name`

### Why `pipe_name` matters

`pipe_name` is part of the runtime routing identity. Lightstream builds a route key as:

- `route_key = <name>::<pipe_name>`

This is important because it prevents collisions when multiple pipelines share the same target `name` (for example, `chat_db` for both `users` and `sessions`).

Rules:

- `pipe_name` is required for every target.
- The route key must be unique across all merged YAML files.
- If two targets resolve to the same route key, startup validation fails.

### Postgres target

```yaml
targets:
  - name: profile_db
    pipe_name: users
    type: postgres
    source_name: user_db
    table: users
    connection_string: ${PROFILE_DB_CONNECTION_STRING}
    conflict_fields: [id]
    transformers: []
```

### HTTP target

```yaml
targets:
  - name: webhook
    pipe_name: user_events
    type: http
    source_name: user_db
    http_details:
      endpoint: https://example.com/hook
      method: POST
      timeout_ms: 15000
      max_concurrent_requests: 6
      headers:
        Content-Type: application/json
        x-api-key: ${WEBHOOK_API_KEY}
      body_template: |
        {
          "id": {{id}},
          "name": {{name}}
        }
```

## HTTP details

`http_details` supports:

- `endpoint` (required, must be valid `http` or `https` URL)
- `method` (required; supported: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`)
- `headers` (optional map)
- `body_template` (optional string template)
- `timeout_ms` (optional; per-request timeout in milliseconds, default: `60000`)
- `max_concurrent_requests` (optional; per-target HTTP concurrency cap, default: `8`)

Template placeholders:

- Use handlebars syntax: `{{field}}`.
- Values are JSON-marshaled during rendering.
- Do not add extra quotes around placeholders unless you explicitly want strings.

## Transformers

Transformers run in order under `targets[].transformers[]`.

Common shape:

```yaml
transformers:
  - type: PickFieldsTransformer
    fields_to_pick: [id, name]
```

### 1) `EntityRenameTransformer`

Renames the record entity (and optionally namespace).

- Config key: `entity_rename_mapping`
- Supports mapping by:
  - full name: `namespace.entity`
  - entity-only: `entity`

```yaml
- type: EntityRenameTransformer
  entity_rename_mapping:
    public.users: crm.customers
    orders: sales_orders
```

### 2) `OmitFieldsTransformer`

Removes fields from `NewPayload` and `OldPayload`.

- Config key: `fields_to_omit`

```yaml
- type: OmitFieldsTransformer
  fields_to_omit:
    - password_hash
    - internal_notes
```

### 3) `PickFieldsTransformer`

Keeps only selected fields in `NewPayload`.

- Config key: `fields_to_pick`
- `OldPayload` is not filtered by this transformer.

```yaml
- type: PickFieldsTransformer
  fields_to_pick:
    - id
    - name
    - created_at
```

### 4) `RenameFieldsTransformer`

Renames field keys in both `NewPayload` and `OldPayload`.

- Config key: `rename_fields_mapping`

```yaml
- type: RenameFieldsTransformer
  rename_fields_mapping:
    name: user_name
    email: user_email
```

### 5) `FieldMappingTransformer`

Maps field names in both payloads (current runtime behavior is equivalent to rename).

- Config key: `field_mapping`

```yaml
- type: FieldMappingTransformer
  field_mapping:
    status: subscription_status
```

### 6) `SleepTransformer`

Pauses processing for a fixed duration.

- Config key: `sleep_time` (milliseconds)

```yaml
- type: SleepTransformer
  sleep_time: 250
```

### 7) `DatabaseLookupTransformer`

Enriches payload by querying another database and merging returned columns into both payloads.

- Config keys:
  - `database_type` (currently `postgres`)
  - `connection_string`
  - `table`
  - `where`
  - `fields`

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

Notes:

- `where` supports placeholders like `{{id}}`.
- Placeholders are converted to query args internally.
- Current runtime behavior on query failure is to continue without enrichment.

### 8) `ValueMapperTransformer`

Sets a field via lookup map, rule evaluation, or default fallback.

- Config keys:
  - `field`
  - `lookup`
  - `rules`
  - `default`

```yaml
- type: ValueMapperTransformer
  field: role
  lookup:
    1: ADMIN
    2: MANAGER
  default: USER
```

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

Rule notes:

- Supported rule type: `match`
- Supports direct match, handlebars-resolved values, and simple `==` expressions.

## Validation summary

Current validation checks include:

- root: `store_connection_string`, non-empty `sources`, non-empty `targets`
- source: `name`, `type`, `mode`
- target: required base fields (`name`, `pipe_name`, `type`, `source_name`); unique `route_key`; HTTP targets validate `http_details`
- transformer: required keys and shape checks by transformer type

Tip: run config validation before startup to catch YAML shape errors early.

## Backfill guide

Use `backfill` when you want to replay existing data from a source into one or more targets.

### Backfill config

Backfill supports:

- `source`: source used for the replay (`postgres` currently).
- `targets`: reused from top-level `targets` (or set directly in backfill-only files).
- `chunk_size`: batch size for fetch/process/load.
- Filtering:
  - `sql_query` for full custom selection.
  - or `date_field` + `from_date`/`to_date` for date-window scans.
- Backfill-only transformer controls:
  - `transformers.mode`: `default | include | exclude | none`
  - `transformers.types`: select by transformer type
  - `transformers.indexes`: select by transformer position (0-based)
  - `transformers.target_overrides[]`: per-target override by `route_key` (`name::pipe_name`)

Example:

```yaml
backfill:
  source:
    name: profile_db
    type: postgres
    mode: notify
    connection_string: ${PROFILE_DB_CONNECTION_STRING}
    tables: [user_tools]
  sql_query: |
    SELECT ut.id, ut."userId", t.name AS "toolName"
    FROM user_tools ut
    JOIN tools t ON t.id = ut."toolId"
  transformers:
    mode: exclude
    types:
      - DatabaseLookupTransformer
  chunk_size: 100
```

### Choosing transformers for backfill

Use this when realtime and backfill need different behavior:

- `default`: use target transformers exactly as configured.
- `none`: run backfill with no transformers.
- `include`: run only selected transformers.
- `exclude`: run all except selected transformers.
- `target_overrides`: fine-tune by specific route key.

Important: `DatabaseLookupTransformer` is not executed by the current backfill processor.  
If a target uses it for realtime ETL, set backfill overrides to exclude it and move enrichment into `sql_query` (for example with `JOIN`/`COALESCE`).

### How backfill runs

For each chunk:

1. Fetch rows from source.
2. Apply configured backfill transformer selection.
3. Process rows per target.
4. Load to targets.
5. Stop when fetch returns no rows.

Execution is concurrent by target type and fails fast on first processing/load error.

### Validation and safety checks

Backfill validation checks include:

- valid `source` and positive `chunk_size`
- valid targets and matching `source_name`
- unique target route keys
- valid backfill transformer override shape:
  - valid `mode`
  - `include`/`exclude` require `types` and/or `indexes`
  - `default`/`none` do not accept selectors
  - override route keys must exist and be unique
  - `indexes` must be `>= 0`

### Operational tips

- Prefer `sql_query` for API targets when source data needs cleanup/normalization.
- Keep batches within endpoint limits (for example `chunk_size: 100`).
- For configs with many source tables, run one backfill file per table/target to keep runs deterministic.
- Read startup logs to confirm effective transformer count per target.
