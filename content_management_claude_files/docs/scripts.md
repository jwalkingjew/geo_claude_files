# Script Reference

All scripts run with `bun run <script>.ts`. Most have a `DRY_RUN` flag at the top.

## Configuration

Scripts read space targets from `src/constants.ts` `SPACES` array. Comment/uncomment entries to control which spaces are processed. The array is ordered by priority (highest first).

## 01_entity_operations.ts

Template script for one-off entity operations. Configure the operation type and entity IDs at the top, then run. Supports:
- Delete entity (single)
- Delete multiple entities (batch — uses shared `OpsBatch` and `deletingIds` set for safe parallel orphan handling)
- Change entity ID
- Merge entities (repoint all references from secondary to primary)
- Migrate property references

## 02.3_export_no_type.ts

Exports all entities with no type assignment from all configured spaces into `no_type_entities.csv`. Uses keyset pagination (`id > lastId`) to avoid offset-based timeout issues on NULL type filters. CSV is consumed by `02_find_duplicates.ts` for cross-checking.

**Config:** `PAGE_SIZE`, `DELAY_BETWEEN_PAGES`, `DELAY_BETWEEN_SPACES`

## 02_find_duplicates.ts

Scans configured spaces for duplicate entities (same name, different IDs) for configurable type categories. Reports data type mismatches for properties. Also cross-checks against `no_type_entities.csv` (from `02.3_export_no_type.ts`) to find typed entities that have no-type twins. Output: console report.

## 02.5_full_space_scan.ts

Cursor-paginated full scan. Reads type list from `../all-types.txt`, scans all entities per type. Also cross-checks against `no_type_entities.csv` for typed-vs-untyped duplicates.

**Usage:** `bun run 02.5_full_space_scan.ts <space>` (crypto, ai, health, podcast)

## 03_merge_duplicates.ts

Auto-merges duplicate entities found by 02. For each duplicate group:
1. Picks survivor by block count first, then backlink count, then property+relation count
2. Saves a pre-merge snapshot to `snapshots/` (JSON with entity data + backlinks for all involved entities)
3. Repoints all references from secondaries to survivor
4. Transfers missing types from secondaries to survivor
5. Skips singleton relations (Avatar, Cover) if survivor already has one
6. Deletes secondaries
7. Publishes ops per space

## 04_merge_duplicates_properties.ts

Same as 03 but specialized for Property entities. Handles data type conflicts: skips merge if primary and secondary have different data types.

## 05_fix_data_types.ts

Two phases:
1. Fix property data type relations (ensure each property points to correct data type entity)
2. Convert mistyped values to correct SDK types based on the property's declared data type

## 06_find_blank_properties.ts

Finds Type entities that have Properties relations pointing to entities with no name. Reports the blank property entity IDs for cleanup.

## 07_fix_stale_relations.ts

Fixes stale relations caused by wrong-space publishing bug in merge operations.

**Phases:**
1. **Detect** - Find entities with relations pointing to deleted targets (no name, not a text block/image/video with content). Uses `entitiesConnection` with server-side filter to narrow candidates.
2. **Resolve** - Match stale relations to correct targets by (fromEntityId, typeId) across spaces. Falls back to exact entityId match, then cross-space search.
3. **Resolve via history** - For remaining unfixable: look up old name/types from version history, search for matching live entity.
4. **Fix** - Generate delete/create ops. Handles: delete stale, create correct, delete wrong-space duplicates.
5. **Orphan cleanup** - Delete stale target entities that have no remaining inbound relations.

**Config:** `DRY_RUN`, `SPACES` in constants.ts

## 08_delete_space_data.ts

Deletes all entities in a space except the home entity and its direct block/type references. Has dry-run mode. Useful for resetting a space.

## 09_undo_merge_project.ts

Reverses the Project type entity merge from `03_merge_duplicates.ts`. Recreates the deleted secondary Project entity with its original name, type, and property relations. Deletes the 10 relations that were added to the main entity during the merge. Also reverts 10 backlinks in space `997f4334...` that were repointed from the secondary to the main.

## 10_fix_leaked_property_relations.ts

Deletes 60 leaked "Properties → Topics" relations that were erroneously created in Root space during `04_merge_duplicates_properties.ts`. These were caused by a cross-space ops bug in `mergeEntities` where backlink migration ops for other spaces leaked into Root's ops batch.

**Background:** The bug was in `entity_ops.ts` — cross-space backlink ops were added to `allOps` (which publishes to `mainSpaceId`) instead of only being routed to their correct space. Fixed in commit `c39a33c`.

## 11_fix_duplicate_subtopics.ts

Finds and deletes duplicate Topics/Subtopics/Broader topics relations across configured spaces. After property merges, some entities ended up with two identical relations to the same target.

**Config:**
- `SPACES_TO_CHECK` — which spaces to scan
- `PROPERTIES_TO_CHECK` — which relation types to deduplicate
- `TEST_ENTITY_ID` — set to an entity ID to test on a single entity before running full scan
- `DRY_RUN` — preview before publishing
