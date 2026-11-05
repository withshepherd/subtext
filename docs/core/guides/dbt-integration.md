# dbt Integration

Import a dbt project into Wren AI to query dbt models through the Wren context layer.

## What Wren Imports

dbt already contains the context an agent needs to query modeled data safely:
model and column descriptions, refs, source definitions, test metadata,
compiled SQL, adapter profile settings, and the physical column list in
`catalog.json`. Wren imports that context so agents do not have to infer model
purpose, join paths, or trusted constraints from table and column names alone.

Wren stores imported dbt context in regular Wren project files:

| dbt input | Wren output | Why it matters |
|-----------|-------------|----------------|
| Active dbt target | Wren connection profile | Reuses the same warehouse connection target dbt uses |
| Model and source nodes | `models/*/metadata.yml` | Preserves model names, table references, descriptions, layers, and columns |
| `relationships` tests | `relationships.yml` | Turns tested dbt refs into explicit Wren join paths |
| `not_null`, `unique`, `accepted_values` tests | Model and column metadata | Gives agents verified constraints and useful filter values |
| Test results | `knowledge/rules/general.md` | Surfaces verified constraints and warnings for agent workflows |
| Model graph and metadata | `knowledge/sql/*.md` | Seeds memory with dbt-aware example questions and SQL |

The generated `wren_project.yml` also keeps a `dbt` binding:

```yaml
dbt:
  project_dir: ../your-dbt-project
  profile: your_dbt_profile
  target: dev
```

`project_dir` points back to the imported dbt project, while `profile` and
`target` record the dbt profile target used for the import. This makes the
Wren project traceable to its dbt source and gives future tooling enough
context to refresh or inspect the original dbt artifacts.

The `dbt` binding stores only project metadata. Credentials stay in Wren
profiles and any environment variables referenced by those profiles.

## Prerequisites

- A dbt project with `dbt_project.yml`
- A dbt profile in `~/.dbt/profiles.yml`, or a custom `profiles.yml`
- Generated dbt artifacts:

```bash
cd your-dbt-project
dbt build
dbt docs generate
```

`dbt docs generate` is required because Wren imports the authoritative column list and types from `target/catalog.json`.

## Import the dbt Profile

Convert the active dbt target into a Wren connection profile:

```bash
wren profile import dbt --project-dir ./your-dbt-project
```

Useful options:

| Flag | Description |
|------|-------------|
| `--project-dir` | dbt project root. Defaults to `.` |
| `--profiles-path` | Custom path to dbt `profiles.yml` |
| `--profile` | dbt profile name override |
| `--target` | dbt target name override |
| `--name` | Destination Wren profile name |
| `--no-activate` | Save the profile without making it active |

Supported adapters include `postgres`, `bigquery`, `snowflake`, `databricks`, `trino`, `clickhouse`, `duckdb`, `mysql`, `redshift`, `spark`, `athena`, `mssql`, and `doris`.

## Import the dbt Project

Generate a Wren project from dbt artifacts:

```bash
wren context import dbt \
  --project-dir ./your-dbt-project \
  --path ./wren-project
```

Preview generated files first:

```bash
wren context import dbt \
  --project-dir ./your-dbt-project \
  --path ./wren-project \
  --dry-run
```

Use `--force` to overwrite files managed by the importer.

Generated files:

| File | Description |
|------|-------------|
| `wren_project.yml` | Project metadata, data source, and dbt binding |
| `models/*/metadata.yml` | Imported dbt models and sources |
| `relationships.yml` | Joins inferred from dbt `relationships` tests |
| `knowledge/rules/general.md` | dbt test summary, verified constraints, and warnings |
| `AGENTS.md` | Agent workflow guidance |
| `knowledge/sql/*.md` | dbt-derived seed NL-SQL pairs for memory indexing |

The importer skips ephemeral models, nodes without catalog columns, and manifest-only columns that are not present in `catalog.json`.

### Exclude Models and Sources by Tag

Use the optional, repeatable `--exclude-tag` flag to omit models and sources
whose resolved tags in `manifest.json` match any supplied tag exactly. This
includes tags inherited through dbt configuration. Without the flag, the
importer keeps its existing selection behavior. `subtext_exclude` is a suggested
convention; the importer has no default exclusion tag.

For example, tag an internal model in a dbt model properties YAML file:

```yaml
models:
  - name: internal_reconciliation
    config:
      tags: [subtext_exclude]
```

Regenerate the dbt artifacts with `dbt build` and `dbt docs generate`, then import
into a clean output directory:

```bash
wren context import dbt \
  --project-dir ./dbt \
  --path ./build/project \
  --exclude-tag subtext_exclude
```

Repeat the flag to exclude resources matching **any** of several tags:

```bash
wren context import dbt \
  --project-dir ./dbt \
  --path ./build/project \
  --exclude-tag subtext_exclude \
  --exclude-tag internal_only \
  --dry-run
```

Both imports and dry runs report the excluded-resource count. Programmatic
imports accept the same selection through
`convert_dbt_project_to_wren_project(..., exclude_tags=["subtext_exclude"])`.

Exclusions apply only to matching resources. Untagged materialized downstream
dbt models remain queryable through their own warehouse tables or views, and
the warehouse handles their dependencies on excluded resources. The importer
omits relationships involving excluded resources and generates query examples
from the retained catalog. Maintained Subtext semantic views or cubes that
reference an excluded semantic model may need adjustment.

Exclusions take effect when you regenerate, build, and deploy the context.
They remove semantic definitions; they do not establish a database authorization
boundary. Database permissions still govern access to the physical relations.

Use a fresh, dedicated import directory for each regeneration so previously
generated files cannot keep excluded models or query examples in the catalog.
The existing `--force` workflow clears the entire `models/` and `views/`
directories and replaces relationships and project metadata, so preserve any
maintained definitions separately. It leaves old `knowledge/sql/` examples and
legacy `queries.yml` files in place. Importing into a fresh directory avoids
those stale files; then restore maintained definitions after reviewing their
references, build the context, and deploy it.

## dbt Test Mapping

Wren imports dbt tests as semantic metadata:

| dbt test | Wren output |
|----------|-------------|
| `not_null` | `not_null: true` on the column |
| `unique` + `not_null` | `is_primary_key: true` and model `primary_key` |
| `accepted_values` | `properties.accepted_values` |
| `relationships` | An entry in `relationships.yml` |

Relationship tests stay in `relationships.yml`; the importer does not stamp relationship dereferences onto FK columns.

## Build and Query

Compile the Wren project:

```bash
wren context build --path ./wren-project
```

Run SQL against imported model names:

```bash
wren --sql "SELECT * FROM fct_orders LIMIT 5"
```

## Memory

If memory is installed, index the imported project:

```bash
wren memory index --path ./wren-project
```

The memory index includes dbt descriptions, layers, test status, accepted values, and seed pairs from `knowledge/sql/*.md`.

## Complete DuckDB Example

```bash
cd jaffle_shop_duckdb
dbt build
dbt docs generate

wren profile import dbt --project-dir .
wren context import dbt --project-dir . --path ../wren-jaffle
wren context build --path ../wren-jaffle
wren --sql "SELECT * FROM fct_orders LIMIT 5"
```

## Troubleshooting

`dbt project file not found`: make sure `--project-dir` points to the directory containing `dbt_project.yml`.

`dbt manifest file not found`: run `dbt build` or `dbt compile`.

`dbt catalog file not found`: run `dbt docs generate`.

`Environment variable 'X' is required`: export variables referenced by `env_var()` in your dbt profile before importing.

Models skipped without columns: regenerate `catalog.json` with `dbt docs generate`, then re-run `wren context import dbt --force`.
