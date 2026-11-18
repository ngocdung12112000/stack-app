---
name: db-migration
description: Make a database schema change in a StackShop service. Covers EF Core migrations for MySQL services (identity, inventory, ordering, payment, shipping, promotion) and index/collection/schema-version changes for MongoDB services (catalog, review, notification, analytics), plus Elasticsearch mapping changes via reindex. Produces the migration bundle used by the K8s migration Job. Use for any new table, column, index, collection or mapping.
---

# Database change

Schemas are documented in `docs/03-data-architecture.md`. Update it in the same change (`sync-docs-adr`).
**Never touch another service's database.** Migrations run as a K8s **Job before rollout**, never on app startup in prod.

## A. MySQL (EF Core + Pomelo)

1. Change the entity + `IEntityTypeConfiguration<T>`. Conventions:
   - `snake_case` table/column names
   - `DATETIME(6)` UTC
   - `DECIMAL(18,2)` for money
   - a `row_version` concurrency token on aggregates that need it
   - explicit indexes for every query the spec lists
2. Create the migration:
   ```bash
   cd services/<svc>
   dotnet ef migrations add <PascalCaseName> -p src/<Svc>.Infrastructure -s src/<Svc>.Api -o Persistence/Migrations
   ```
3. Review the generated SQL (`dotnet ef migrations script <prev> <new> --idempotent`). Check for:
   - table rewrites / locks on big tables
   - a data-loss warning (drop/rename)
   - a missing default for NOT NULL columns
4. **Zero-downtime rule (expand → contract)**, because the old and new pods run side by side during a rolling update:
   - Add a column as nullable or with a default → deploy code that writes it → backfill → make it NOT NULL in a later migration.
   - Rename = add the new column + dual write + backfill + switch reads + drop the old one later. Never rename in place.
   - Drop a column/table only after no deployed version uses it.
5. Build the bundle for the Job: `dotnet ef migrations bundle -p src/<Svc>.Infrastructure -s src/<Svc>.Api --self-contained -r linux-x64 -o efbundle` (the Dockerfile has a `migrator` target or image).
6. Test: the integration tests apply migrations on a Testcontainers MySQL from scratch. Also test upgrading from the previous migration if there's a data backfill.

## B. MongoDB

- There are no migrations for the schema itself. Documents carry a `schemaVersion` when the shape changes, and readers handle old versions (lazy upgrade on write) or a one-off backfill job is written.
- Indexes: declared in code (`CreateIndexesAsync`, idempotent) at startup or in a migrator command. Create big indexes in the background and note it in the PR.
- Renaming fields: add the new one → dual write → backfill script (in `tools/` or the service's migrator command) → stop reading the old one.
- Test with Testcontainers Mongo **as a replica set** (transactions).

## C. Elasticsearch mapping

A mapping change on an existing field needs a **new index version** + alias swap (`docs/03-data-architecture.md` §5.2). Bump `products-v{n+1}` in the index template code, use the reindex endpoint, and swap the alias. Never update mappings in place for changed field types.

## Handoff

**devops-engineer:** the migration Job image/command changed (bundle), any long-running migration that needs a longer Job `activeDeadlineSeconds`, and new DB privileges if a new schema object type is needed.
