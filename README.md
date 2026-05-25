# TYPO3 Workspace Overlay Patch

Composer patch bundle for TYPO3 v14.3 LTS. Fixes two related backend regressions when the user works in a workspace:

1. Field-filtered queries (record-list search, live search, suggest wizard, slug uniqueness) silently miss values that exist only in a workspace draft — the default `WorkspaceRestriction` excludes regular workspace versions, so overlay never sees them.
2. Workspace rows that reach `BackendUtility::workspaceOL()` / `PageRepository::versionOL()` directly are not normalized, so consumers expecting the live uid get the offline uid instead.

The fix adds a new `QueryBuilder::addWorkspaceRestriction()` helper, a central decision helper `WorkspaceRestriction::shouldIncludeAllVersionedRecordsForBackendQuery()`, and two normalization blocks in the overlay functions. Everything is gated by **one TSconfig key** so the patch behaves as a clean on/off switch.

## Install

```sh
composer require webconsulting/typo3-workspace-overlay-patch
```

This package only carries patches — no PHP runtime code. The patches are applied by [`cweagans/composer-patches`](https://github.com/cweagans/composer-patches) v2 against `typo3/cms-core` and `typo3/cms-backend`.

Your root `composer.json` must allow the plugin:

```json
{
  "config": {
    "allow-plugins": {
      "cweagans/composer-patches": true
    }
  }
}
```

## The Switch

Default behavior: **on** for any backend user in a non-live workspace. To disable, set User or Page TSconfig:

```
options.workspaces.includeAllVersionedRecordsInQueries = 0
```

When the switch is off, the patch is fully dormant: search uses the vanilla `WorkspaceRestriction` and the overlay normalizers fall through to the original core logic.

## What Gets Patched

| Half | File | What it does |
| --- | --- | --- |
| Query | `typo3/cms-backend/Classes/RecordList/DatabaseRecordList.php` | Include all workspace rows in record-list search, distinct count by live uid, dedupe after overlay |
| Query | `typo3/cms-backend/Classes/Search/LiveSearch/DatabaseRecordProvider.php` | Same broadening for live-search records |
| Query | `typo3/cms-backend/Classes/Search/LiveSearch/PageRecordProvider.php` | Same broadening for live-search pages |
| Query | `typo3/cms-backend/Classes/Form/Wizard/SuggestWizardDefaultReceiver.php` | Suggest wizard queries see workspace-only labels |
| Query | `typo3/cms-core/Classes/DataHandling/SlugHelper.php` | Slug uniqueness sees workspace-only slug values |
| Overlay | `typo3/cms-backend/Classes/Utility/BackendUtility.php` | `workspaceOL()` normalizes direct workspace rows to live uid, drops delete placeholders |
| Overlay | `typo3/cms-core/Classes/Domain/Repository/PageRepository.php` | `versionOL()` same normalization in frontend / domain repository |
| API | `typo3/cms-core/Classes/Database/Query/QueryBuilder.php` | New `addWorkspaceRestriction()` developer helper |
| API | `typo3/cms-core/Classes/Database/Query/Restriction/WorkspaceRestriction.php` | New `shouldIncludeAllVersionedRecordsForBackendQuery()` central decision helper |

## Compatibility

- TYPO3 14.3 LTS (currently pinned to 14.3.1; rebuild against newer patch releases as needed).
- PHP 8.2+.
- `EXT:workspaces` must be installed for the patch to do anything (the gate returns `false` otherwise).

## Documentation

- [`docs/typo3-workspace-query-overlay-explained.html`](docs/typo3-workspace-query-overlay-explained.html) — full long-form explanation with file-by-file walkthrough, test results, and the gate diagram.
- [`docs/typo3-workspace-query-overlay.md`](docs/typo3-workspace-query-overlay.md) — original analysis report (problem scan, ten risk areas, generic solution).

## License

GPL-2.0-or-later, matching TYPO3 core.

## Status

Verified against TYPO3 14.3.1 with PHP 8.3.30, MariaDB 10.11. Five upstream functional tests pass (see the HTML doc for the reproduction recipe). Not yet submitted upstream as a Forge / Gerrit change.
