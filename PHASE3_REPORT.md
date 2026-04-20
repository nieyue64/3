# Phase 3 Report: Resilience & Worker Rebuild

## Completion: 90%

## What Was Done

### 1. BlockRegistry with Palette Mapping (New - Critical)
- Created `window.TU.BlockRegistry` with deterministic ID allocation
- `allocate(name, preferredId)` - Allocates an ID for a named block type, returns existing if already allocated
- `getPalette()` - Returns name->id mapping for save serialization
- `applyPalette(savedPalette)` - Compares saved palette with current, returns remap table if IDs differ
- `getName(id)` - Reverse lookup for debugging
- `freeze()` - Prevents further allocations after init

### 2. Integrated BlockRegistry with Existing Code
- **tileLogic section**: WIRE_OFF/ON, SWITCH_OFF/ON, LAMP_OFF/ON now use `BlockRegistry.allocate()` with preferred IDs (200-205)
- **v9_biomes section**: PUMP_IN, PUMP_OUT, PLATE_OFF, PLATE_ON now use `BlockRegistry.allocate()` with preferred IDs (206-209)
- `allocId()` function updated to accept name parameter and delegate to BlockRegistry
- Legacy fallback preserved if BlockRegistry is not available

### 3. Save Palette Mapping
- `buildFullPayload()` now includes `blockPalette` field in save headers
- Contains the full name->id mapping from BlockRegistry
- On load, `applyPalette()` can detect ID drift and return a remap table
- Backward compatible: old saves without palette still load correctly

### 4. StorageAdapter (New)
- Created `window.TU.StorageAdapter` with explicit degradation flow:
  - `detect()` - Tests LS and IDB availability, sets mode
  - `write(key, value, opts)` - Writes with LS->IDB fallback, reports QuotaExceeded
  - `read(key, opts)` - Reads with fallback chain
  - `getMode()` - Returns current mode: 'full', 'idb-only', 'ls-only', 'degraded'
- QuotaExceededError now triggers mode change and emits event
- IDB failures tracked through GlobalErrorBoundary

### 5. Worker Timeout Detection
- `WorldWorkerClient.generate()` now uses `GlobalErrorBoundary.trackWorkerMessage()` with 60s timeout
- If worker hangs, the generate Promise is rejected instead of hanging forever
- Timeout cleared on successful 'done' message receipt
- User gets toast notification about worker hang

### 6. Worker toString() Rebuild (Deferred)
- The `fn.toString()` pattern for worker source code is deeply embedded in `_buildWorkerSourceParts()`
- Full replacement with constant templates would require rewriting the entire worker assembly
- This is a large, high-risk change best done as a separate focused task
- Current toString() pattern works correctly; the risk is only with future code minification

## Files Changed
- `part3_game_single (6) (9)(9)(2)(6).html` - Main game file

## Backup
- `part3_phase3_resilience.html` - State after Phase 3

## Risk Assessment
- **Low risk**: BlockRegistry is additive - existing code still works if registry missing (fallback)
- **Low risk**: StorageAdapter is new utility, not yet wired into main save path (incremental)
- **Medium risk**: Worker timeout may reject generate prematurely on slow devices - 60s is generous
- **Deferred**: Worker toString() rebuild is too high-risk for this batch
