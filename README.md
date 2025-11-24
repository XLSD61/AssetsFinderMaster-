# AssetsFinderMaster — Unused & Used Assets Finder Toolkit

**Filename suggestion:** `AssetsFinderMaster_Documentation_v1.0.pdf`

---

## Table of Contents

1. Introduction ........................................... 1
2. Package Contents ..................................... 2
3. Requirements & Compatibility ........................ 3
4. Installation (Step-by-step) .......................... 4
5. Quick Start Guide ................................... 6
6. Feature Overview .................................... 8
   6.1 Scan Scopes
   6.2 Dependency Analysis (Scenes, Prefabs, Materials)
   6.3 Exclusions & Custom Folders
   6.4 Safe vs. Dynamic Assets (Resources / StreamingAssets)
   6.5 Single-asset Actions: Locate / Quarantine / Delete
   6.6 Bulk Actions: Bulk Quarantine / Bulk Delete (with confirmations)
   6.7 Progress Reporting & Refreshing
   6.8 Styling & UI Options
7. Script Reference ................................... 15
   7.1 Namespace: `AssetsFinderMaster`
   7.2 `UnusedAssetsFinder` (EditorWindow) — public API & internal behavior
   7.3 `UsedAssetsFinder` (if present) — public API & internal behavior
   7.4 Utility methods and editor helpers
8. Common Use Cases / Examples ......................... 28

   * Project cleanup pass
   * Pre-submission sanity checks
   * CI / build-time validation (editor script hooks)
   * Quarantine workflow for large teams
9. API Behavior & Implementation Notes .................. 34

   * AssetDatabase usage and GUID handling
   * File system moves and meta handling
   * Editor dialog behavior and defensive confirmations
   * Progress bar and cancel safety
10. Troubleshooting & Known Issues ..................... 38

    * Platform/editor differences for dialogs
    * File permission / path issues when quarantining
    * False positives due to runtime dynamic loads
11. Author / Support ................................... 41
12. License ............................................ 42

---

## 1. Introduction

AssetsFinderMaster is a Unity Editor toolkit that helps you find, inspect, quarantine and delete **unused** and **used** assets in your project. It includes two main Editor windows (or tools): **UnusedAssetsFinder** and **UsedAssetsFinder** (UsedAssetsFinder is documented here if included in the package). The package focuses on safety and workflow: it separates assets that might be dynamically loaded, provides multi-stage confirmations for bulk operations, and offers quarantine capabilities to move files outside the project instead of immediate permanent deletion.

This documentation is intended for Asset Store submission and for developers integrating the tool into their workflow. It covers installation, quick start, full API details, examples, implementation notes and troubleshooting.

---

## 2. Package Contents

```
/Assets/AssetsFinderMaster/
  /Runtime/ (optional runtime helpers, if any)
  /Editor/
    UnusedAssetsFinder.cs         // Main EditorWindow for unused asset discovery
    UsedAssetsFinder.cs           // (Optional) EditorWindow for locating assets that are referenced
    AssetsFinderDocsLink.cs       // Menu link: Tools > AssetsFinderMaster > Documentation
  README.md
  LICENSE
```

Important files:

* `UnusedAssetsFinder.cs` — full-featured EditorWindow scanning for unused assets by type, with per-item and bulk operations (locate, quarantine, delete).
* `UsedAssetsFinder.cs` — (optional) complementary window that focuses on assets known to be used / referenced and their references.
* `AssetsFinderDocsLink.cs` — adds a menu entry to open documentation.

---

## 3. Requirements & Compatibility

* **Unity Editor:** Recommended 2019.4 LTS or newer; minimum supported 2018.4.
* **Scripting runtime:** .NET 4.x compatibility is recommended for best Editor utility support.
* **Platforms:** Editor-only tool. Works across OSes supported by Unity Editor (Windows, macOS, Linux).
* **Permissions:** File move/delete operations require write permission to project folder and target quarantine destination.
* **Notes:** The tool uses `AssetDatabase`, `EditorUtility` dialogs, and file system APIs — behavior may differ slightly between Unity versions. Test on the target Unity version before wide deployment.

---

## 4. Installation (Step-by-step)

### 4.1 Via Package (Unity Package / Asset Store)

1. Download the package from the Asset Store or your distribution.
2. In Unity: `Assets > Import Package > Custom Package...` and choose the downloaded `.unitypackage`.
3. In the Import dialog press **Import** and allow the editor to compile.

### 4.2 Manual copy

1. Copy the `AssetsFinderMaster` folder into your project under `Assets/`.
2. Open Unity; let scripts compile.

### 4.3 Verify installation

1. In Unity Editor menu bar: `Tools > AssetsFinderMaster`. There should be options such as `Open Unused Assets Finder` or `Documentation`.
2. Confirm `Assets/AssetsFinderMaster/Editor/UnusedAssetsFinder.cs` exists.

---

## 5. Quick Start Guide

### 5.1 Open the window

* `Tools > AssetsFinderMaster > Find Unused Assets` — opens **UnusedAssetsFinder**.

### 5.2 Basic scan (defaults)

1. Choose scan scope: `Assets` (default) or `Custom Folders`.
2. Select asset types to include (check/uncheck types).
3. Optional: Add paths to **Excluded Folders** (recommended to exclude `Resources/` and `StreamingAssets/`).
4. Click **Find Unused Assets**. The tool will scan scenes, prefabs, materials and gather used asset dependencies then list candidate unused assets by type.

### 5.3 Inspect / single actions

* For any listed asset row use:

  * **Locate** — selects and pings the asset in Project window.
  * **Quarantine** — moves the asset (and .meta) to a configured quarantine folder outside the project.
  * **Delete** — moves the asset to Trash (using `AssetDatabase.MoveAssetToTrash`).

### 5.4 Bulk operations

* Use **Bulk Quarantine** or **Bulk Delete** in the top toolbar to process all found assets. The tools include:

  * Special handling for assets in `Resources/` or `StreamingAssets` (they are separated as *dynamic*).
  * A `DisplayDialogComplex` that gives choices; implemented so the first button is always Cancel to treat window close (X) as Cancel across Unity versions.
  * A secondary **final confirmation** dialog for extra safety.
  * Progress bar during processing.

---

## 6. Feature Overview

### 6.1 Scan Scopes

* **AssetsFolderOnly:** scans the `Assets/` root.
* **AssetsAndPackages:** includes packages (if implemented).
* **CustomFolders:** scan only user-selected folders.

### 6.2 Dependency Analysis

* Scans Scenes (`t:Scene`), Prefabs (`t:Prefab`) and Materials (`t:Material`) within the chosen scope.
* Gathers dependencies using `AssetDatabase.GetDependencies(path, true)` to build a `usedAssets` set.
* Assets referenced by scanned content are excluded from "unused" results.

### 6.3 Exclusions & Custom Folders

* Excluded folders (drag & drop UI) prevent false positives for folders such as third-party plugins, streaming data, or special pipelines.
* Custom folders allow focused scans (e.g., `Assets/Art/Characters`).

### 6.4 Safe vs. Dynamic Assets

* Any asset inside `/Resources/` or `/StreamingAssets/` is flagged as **dynamic** and separated from **safe** assets.
* Bulk operations prompt special warnings when dynamic assets are present.

### 6.5 Single-asset Actions

* **Locate:** uses `AssetDatabase.LoadAssetAtPath` and `EditorGUIUtility.PingObject`.
* **Quarantine:** moves the file to a user-specified quarantine folder (outside the project root). Also removes .meta. Uses safe unique naming if collisions occur.
* **Delete:** uses `AssetDatabase.MoveAssetToTrash` to move to OS Trash/Recycle Bin.

### 6.6 Bulk Actions & Safety

* Bulk operations iterate assets, show progress via `EditorUtility.DisplayProgressBar`, and log success/warnings.
* Special dialog ordering & final confirmation added to avoid X-close mistakes in `DisplayDialogComplex`.
* Errors during processing are caught and presented to the user.

### 6.7 Progress Reporting & Refreshing

* After operations, `AssetDatabase.Refresh()` is called; UI offers delayed re-scan (`EditorApplication.delayCall`) to avoid GUI race conditions.
* `shouldRefreshAssets` flag ensures the UI refreshes once.

### 6.8 Styling & UI

* Custom GUIStyles for asset rows and buttons (locate, quarantine, delete) to improve readability.
* Warning icon for assets in Resources/StreamingAssets with explanatory tooltip.

---

## 7. Script Reference

### 7.1 Namespace

All editor code lives in the global (or optionally `AssetsFinderMaster`) namespace. Example suggestion:

```csharp
namespace AssetsFinderMaster.Editor { ... }
```

### 7.2 `UnusedAssetsFinder` (EditorWindow)

**Class:** `UnusedAssetsFinder : EditorWindow`

**Purpose:** Scans project (or selected folders) and lists assets that appear to be unused based on dependency analysis. Provides per-asset actions and bulk operations.

**Public entry point:**

```csharp
[MenuItem("Tools/AssetsFinderMaster/Find Unused Assets")]
public static void ShowWindow() { ... }
```

**Key fields (summary):**

* `enum SearchScope { AssetsFolderOnly, AssetsAndPackages, CustomFolders }`
* `enum SortMode { None, Size, Name, DateModified }`
* `string quarantineFolderPath`
* `List<DefaultAsset> customFolders`
* `List<DefaultAsset> excludedFolders`
* `Dictionary<string, List<string>> unusedAssetsByType`
* `Dictionary<string, GUIStyle> assetTypeStyles`
* `Dictionary<string, bool> assetTypeToggles`

**Important methods & behavior:**

* `void FindUnusedAssets()`

  * Builds `usedAssets` by scanning Scenes, Prefabs and Materials in the selected search scope.
  * Iterates configured `assetTypeToggles` and finds GUIDs via `AssetDatabase.FindAssets("t:Type", searchInFolders)`.
  * Skips editor scripts (`/Editor/`) for `MonoScript`.
  * Filters `IsPathExcluded(path)` and `usedAssets.Contains(path)` before adding to `unusedAssetsByType`.
  * Calls `ApplySorting()` and clears progress bar.

* `void ApplySorting()`

  * Sorts asset lists by selected `SortMode` and `sortAscending`.

* `void BulkDeleteAssets()`

  * Collects all unused asset paths.
  * Calls `SeparateAssets` to divide dynamic vs safe.
  * Shows `DisplayDialogComplex` with Cancel as first button.
  * Shows a **Final Confirmation** `DisplayDialog` (two-button) before processing.
  * Calls `ProcessBulkDeletion(finalAssetsToProcess)`.

* `void ProcessBulkDeletion(List<string> assetsToProcess)`

  * Loops and `AssetDatabase.MoveAssetToTrash(assetPath)` with progress bar.
  * Logs results and shows completion dialog. Handles exceptions and always clears progress bar.

* `void BulkQuarantineAssets()` and `void ProcessBulkQuarantine(List<string>)`

  * Similar flow to bulk delete but moves files to a quarantine folder chosen via `EditorUtility.OpenFolderPanel`.
  * Ensures unique dest names, moves .meta handling, `AssetDatabase.Refresh()` at the end.

* `void QuarantineAsset(string assetPath)`

  * Single-asset quarantine with folder selection if not configured.

* `void DeleteAsset(string assetPath)`

  * Single-asset deletion with `DisplayDialog` confirm and `AssetDatabase.MoveAssetToTrash`.

* `bool IsPathExcluded(string assetPath)`

  * Returns true if asset path starts with any excluded folder path.

* `List<string> GetAllUnusedAssetPaths()` and `(List<string> dynamicLoadAssets, List<string> safeAssets) SeparateAssets(List<string>)`

  * Helpers for bulk flow.

**UI notes:**

* `DrawFolderList()` provides Add and Remove (X button) for custom/excluded folders. The removal X is a deliberate single-action button.

**Important implementation notes documented inline:**

* First button in `DisplayDialogComplex` is Cancel to neutralize X-close behavior on some Unity versions.
* Final confirmation step added to avoid accidental bulk operations.

### 7.3 `UsedAssetsFinder` (optional)

**Purpose:** Complementary tool that lists assets that are referenced and where they are referenced (inverse dependency view). Implementation patterns are similar (use `AssetDatabase.FindAssets`, `AssetDatabase.GetDependencies`, `AssetDatabase.GetAssetPath`), but the focus is on answering "Where is this asset used?" rather than "Is this asset unused?"

**Suggested methods:**

* `List<string> FindReferences(string assetPath)` — returns referencing asset paths.
* UI: select asset, click **Find References**, show collapsible list of references.

### 7.4 Utility methods & Editor helpers

* GUI style creation helpers (texture generation for backgrounds).
* Safe file move utilities with unique-name conflict resolution.
* Logging helpers standardized to `[AssetsFinderMaster]` prefix.

---

## 8. Common Use Cases / Examples

### Project cleanup pass

1. Exclude `Resources/` and `StreamingAssets`.
2. Scan `Assets` for common asset types.
3. Review `Unused Assets by Type` foldouts.
4. For small batches: `Quarantine` first; test project; then `Bulk Delete` safe items.

### Pre-submission sanity checks

* Run the tool, quarantine suspicious assets, run build and QA. Restore from quarantine if necessary.

### CI integration (advanced)

* Create an Editor script that calls `FindUnusedAssets()` headless and writes results to a JSON — useful for automated reports (note: `EditorUtility.DisplayDialog` calls must be avoided in headless mode).

---

## 9. API Behavior & Implementation Notes

### AssetDatabase and GUIDs

* The tool uses asset paths returned by `AssetDatabase.GUIDToAssetPath` as primary identifiers. When moving files out of the project, .meta files are removed — this intentionally breaks internal asset references until restored from quarantine.

### File system moves & meta

* Quarantine moves the source file and deletes `.meta`. Restoring requires manual reinsertion of the file and reimport/regeneration of meta if needed. Document this workflow clearly for team use.

### Dialog robustness

* Because `EditorUtility.DisplayDialogComplex` behavior on window-close varies across versions/platforms, the code places Cancel as the first option and supplies a final two-button `DisplayDialog` as an extra safety net.

### Progress & cancellation

* Progress bars are used for long operations. If a long running process is interrupted (e.g., Unity closed), behavior follows standard OS file system rules — no guaranteed rollback. Keep backups.

---

## 10. Troubleshooting & Known Issues

### Dialog X-close treated as confirm on some Unity versions

* Mitigation: Cancel is placed as first button in complex dialogs and a final confirmation is required. If you still see accidental confirmations, consider switching to a custom modal `EditorWindow` implementation.

### Permission / file locked errors when quarantining

* Ensure Unity and your user account have write permission to both project and quarantine destination. Antivirus may lock files on some platforms.

### False positives: runtime dynamic loads

* Always exclude `Resources/` and `StreamingAssets`. Also be aware of custom loading pipelines (Addressables, reflection-based loads). When in doubt, quarantine not delete.

### Restoring quarantined files

* Quarantined files have no `.meta` in the quarantine folder. Restoring to `Assets/` will cause Unity to generate new `.meta` files (and new GUIDs) unless you also moved .meta. Document your desired restore workflow.

---

## 11. Author / Support

**Author:** Dorukhan
**Support & Issues:** `dorukhanozaksu@hotmail.com` or link to repository issues.
When reporting: include Unity version, OS, exact reproduction steps and any editor console logs.

---

## 12. License

Include the LICENSE file in the package root. Typical choices:

* **MIT** — permissive with attribution.
* Or a custom EULA for Asset Store distribution.

---

## Appendix — Best Practices & Recommendations

* Always run `Find Unused Assets` on a branch or a copy of the project before mass-deleting files.
* Use **Quarantine** for non-obvious removals; keep quarantine archived for a short period before deleting.
* Add `AssetsFinderMaster` as part of your repository’s release checklist for post-refactor cleanup.
* For very large projects: limit scan to subfolders or run scans incrementally to avoid long Editor stalls.

---


