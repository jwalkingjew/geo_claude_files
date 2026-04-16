# Content Management Scripts

Bun-based scripts for managing entities, relations, and properties across Geo Protocol knowledge graph spaces.

## Quick Reference

- **Runtime:** `bun run <script>.ts`
- **API:** GraphQL at `https://testnet-api.geobrowser.io/graphql` (see `src/functions.ts`)
- **SDK:** `@geoprotocol/geo-sdk` for graph operations and publishing
- **Auth:** `.env` requires `PK_SW` (smart wallet private key) and `DEMO_SPACE_ID`

## Critical: API Pagination

**The `entities`, `relations`, and `values` queries have a hard offset cap at 1000.** Offset-based pagination will 400 error beyond this.

**Always use cursor-based connection endpoints:**
- `entitiesConnection` / `relationsConnection` / `valuesConnection`
- Use `first`, `after`, `pageInfo { hasNextPage endCursor }`
- Prefer `nodes { ... }` over `edges { node { ... } }` for simpler access
- **`entitiesConnection` uses top-level `spaceId` arg; `relationsConnection`/`valuesConnection` use `filter: { ... }`**
- See `docs/pagination.md` for patterns and examples

## Critical: Connection Filters

Complex nested filters on connection endpoints can cause `INTERNAL_SERVER_ERROR`. Always scope relation filters with `spaceId` to limit the search space. See `docs/pagination.md` for details.

## Critical: Cross-Space Ops in entity_ops.ts

When functions in `entity_ops.ts` handle cross-space operations (backlink migration, property reference migration), the ops MUST be routed to the correct space via `publishOrBatch(ops, ..., spaceId, opsBatch)`. **Never add cross-space ops to a shared `allOps` array** that later gets published to a different space — this causes ops to be double-published to the wrong space. Functions that publish per-space internally (like `migratePropertyReferences`, `updateDataBlockFilters`) should have their return values discarded by the caller, not re-accumulated.

## Critical: deleteEntity Batch Awareness

When deleting multiple entities in a batch, pass a shared `deletingIds: Set<string>` to all `deleteEntity` calls. This prevents:
- **Infinite recursion** on cyclic relations (A→B→C→A)
- **Duplicate orphan cleanup** where sibling deletes both try to clean the same target
- Orphan checks incorrectly counting relations from entities that are themselves being deleted

`deleteEntity` also deletes **incoming relations** (backlinks pointing to the entity), not just outgoing ones.

## Scripts

| Script | Purpose |
|--------|---------|
| `01_entity_operations.ts` | Template for entity CRUD (delete, batch delete, change ID, merge, migrate) |
| `02_find_duplicates.ts` | Find duplicate entities across spaces + no-type cross-check |
| `02.3_export_no_type.ts` | Export all no-type entities from all spaces to CSV |
| `02.5_full_space_scan.ts` | Full cursor-paginated space scan for duplicates |
| `03_merge_duplicates.ts` | Auto-merge duplicate Type/Property entities |
| `04_merge_duplicates_properties.ts` | Merge duplicate Property entities with data type handling |
| `05_fix_data_types.ts` | Fix property data type relations and convert mistyped values |
| `06_find_blank_properties.ts` | Find Type entities with blank property names |
| `07_fix_stale_relations.ts` | Fix stale relations from wrong-space publishing bug |
| `08_delete_space_data.ts` | Delete all entities in a space (dry-run supported) |
| `09_undo_merge_project.ts` | Undo the Project type merge (recreate secondary, revert backlinks) |
| `10_fix_leaked_property_relations.ts` | Delete 60 leaked Properties→Topics relations in Root from property merge bug |
| `11_fix_duplicate_subtopics.ts` | Find/delete duplicate Topics/Subtopics relations across spaces |

## Architecture

- `src/constants.ts` - Space IDs, type IDs, property IDs (see `knowledge-graph-ontology.md` for full registry)
- `src/entity_ops.ts` - Core entity operations (merge, change ID, migrate references, delete with orphan cleanup)
- `src/functions.ts` - GraphQL client (`gql` with exponential backoff), publishing (`publishOps`), membership checks
- `snapshots/` - Pre-merge snapshots saved automatically during non-dry-run merges (for recovery)

## Deeper Docs

- `docs/pagination.md` - Pagination patterns, connection endpoints, filter gotchas
- `docs/scripts.md` - Detailed script behavior, phases, and configuration
- `knowledge-graph-ontology.md` - Full ontology spec (types, properties, data types)
