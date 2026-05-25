# TYPO3 Workspace Query Overlay Analysis

Date: 2026-05-21

## Problem

TYPO3 workspace overlay works only after a candidate row has been selected. Queries that filter on mutable record fields can miss workspace-only values because the default `WorkspaceRestriction` returns live rows plus new and move placeholder rows, but not the changed offline row for an existing record.

Forge issue 109577 fixed one ordering bug in `DatabaseRecordList`: rows are now overlaid before resolved `Record` objects are created. The remaining bug class is different: the query can exclude the workspace row before overlay is ever called.

Example:

1. Live row `uid=1`, `header="Live title"`, `t3ver_wsid=0`.
2. Workspace row `uid=42`, `t3ver_oid=1`, `header="Draft title"`, `t3ver_wsid=1`.
3. Backend search for `Draft title`.
4. Default workspace restriction excludes `uid=42`, so overlay never sees the matching row.

## Scan

All installed `vendor/typo3/cms-*` packages were scanned.

- TYPO3 CMS packages scanned: 34
- `WorkspaceRestriction` occurrences: 78
- `workspaceOL()` / `versionOL()` overlay occurrences: 108

Ten representative risk areas:

1. `typo3/cms-backend/Classes/RecordList/DatabaseRecordList.php`
   Search filters run before overlay. Patched to include all workspace rows only for search, overlay first, then de-duplicate by live uid.
2. `typo3/cms-backend/Classes/Search/LiveSearch/DatabaseRecordProvider.php`
   Live search already used include-all unconditionally. Patched to route through the new API and TSconfig switch.
3. `typo3/cms-backend/Classes/Search/LiveSearch/PageRecordProvider.php`
   Same live-search issue for pages. Patched through the new API and TSconfig switch.
4. `typo3/cms-backend/Classes/Form/Wizard/SuggestWizardDefaultReceiver.php`
   Suggest search filters labels before overlay. Patched to include all workspace rows when enabled.
5. `typo3/cms-core/Classes/DataHandling/SlugHelper.php`
   Slug uniqueness is a create/update path. A draft-only slug could be missed. Patched to include all workspace rows and reuse existing overlay reduction.
6. `typo3/cms-core/Classes/Routing/PageSlugCandidateProvider.php`
   Already has an include-all query for one candidate path. The overlay normalization added here makes directly selected offline rows safe.
7. `typo3/cms-seo/Classes/Widgets/Provider/PagesWithoutDescriptionDataProvider.php`
   Field filters on `description`, `canonical_link`, and `no_index` can still be stale if the draft value is the only match. Covered by the generic overlay normalization, but the query should use the new helper if this widget needs workspace draft-field search semantics.
8. `typo3/cms-recycler/Classes/Domain/Model/DeletedRecords.php`
   Label filter runs before overlay. It should use the new helper when recycler is expected to find draft-only labels.
9. `typo3/cms-info/Classes/Controller/PageInformationController.php`
   Information views query workspace-aware page rows and then derive state. Direct offline-row normalization reduces duplicate/wrong-uid risk if include-all is adopted there.
10. `typo3/cms-info/Classes/Controller/TranslationStatusController.php`
    Translation status queries count and compare workspace-aware rows. This is another candidate for the helper if filters/counts need draft-field semantics.

## Generic Solution

The patch adds a small QueryBuilder API:

```php
$queryBuilder->addWorkspaceRestriction($workspaceId, $includeAllVersionedRecords);
```

This replaces an existing workspace restriction and makes the intent explicit. Callers that filter on editable fields can opt into querying live plus all versioned workspace rows, then run normal workspace overlay/reduction.

The global backend decision is centralized in:

```php
WorkspaceRestriction::shouldIncludeAllVersionedRecordsForBackendQuery()
```

It returns true only when:

- `EXT:workspaces` is loaded.
- A backend user exists.
- The backend user is in a non-live workspace.
- User TSconfig does not disable the behavior.

Disable switch:

```typoscript
options.workspaces.includeAllVersionedRecordsInQueries = 0
```

## Overlay Normalization

`BackendUtility::workspaceOL()` and `PageRepository::versionOL()` now handle rows that are already direct workspace rows from an include-all query:

- Delete placeholders become `false`.
- Existing-record versions get normalized back to the live uid.
- `_ORIG_uid` stores the offline uid.
- Move pointers also keep `_ORIG_pid` when possible.

This keeps existing overlay consumers stable when the query returns an offline row directly instead of the live row.

## Performance

The default live workspace path is unchanged. Include-all is active only for backend workspace queries where the caller uses the helper or the patched search paths use the central switch.

The record-list count query uses `COUNT(DISTINCT CASE WHEN t3ver_oid > 0 THEN t3ver_oid ELSE uid END)` only for search in a non-live workspace. Normal list views keep `COUNT(*)`.

## Tests Added

- QueryBuilder API parity with `WorkspaceRestriction(..., true)`.
- `BackendUtility::workspaceOL()` normalizes direct workspace rows.
- `PageRepository::versionOL()` normalizes direct workspace rows.
- Record list search finds a value that exists only in the workspace row.
- Slug uniqueness sees a slug value that exists only in the workspace row.

