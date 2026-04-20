# Phase 2 Report: Pipeline & API Isolation

## Completion: 85%

## What Was Done

### 1. Scoped Canvas Optimizer to Game Canvas Only (Critical)
- **Before**: `HTMLCanvasElement.prototype.getContext` was globally hijacked, affecting ALL canvas elements (weather overlays, offscreen buffers, minimap, chunk caches, worker render canvases). This broke V8's prototype chain optimization for every canvas in the page.
- **After**: Created `window.TU.CanvasOptimizer.optimize(ctx)` utility that applies the same alpha/fillStyle deduplication optimization to a **specific** context.
- Applied automatically on `game:init:post` to only the main game renderer context.
- All other canvases (weather FX, minimap, chunks, profiler, etc.) now use native unmodified `getContext()`.
- **Impact**: Eliminates JIT deoptimization caused by modified prototype chain on every canvas context creation.

### 2. PostFX Pipeline Infrastructure (New)
- Added `Renderer.postFxPipeline` array and supporting methods:
  - `registerPostFxStage(name, order, fn)` - Register a named stage at a given order
  - `removePostFxStage(name)` - Remove a stage
  - `runPostFxPipeline(time, depth01, reducedMotion)` - Execute all stages in order
- Pipeline is sorted by order number (lower = earlier):
  - Order 10: Base effects (vignette, grain, warm/cool)
  - Order 20: Weather tint
  - Order 30: Underwater fog
- Each stage is wrapped in try-catch to prevent one failed stage from killing the whole pipeline.
- **Status**: Infrastructure ready. Patches can incrementally migrate to use `registerPostFxStage` instead of wrapping `applyPostFX`. Full migration deferred to reduce risk.

### 3. Weather State (Deferred)
- `game.weather` is the canonical weather object, but `window.AppServices.get('weatherFx')` is a separate derived state for post-FX params.
- Full unification deferred - it's used in 7+ locations across both render and update paths.
- Current architecture works correctly; unification is a nice-to-have, not a bug fix.

## Files Changed
- `part3_game_single (6) (9)(9)(2)(6).html` - Main game file

## Backup
- `part2_phase2_pipeline_isolation.html` - State after Phase 2

## Risk Assessment
- **Low risk**: Canvas optimizer change is purely scoped - removes global side effect
- **No risk**: Pipeline infrastructure is additive only (no existing behavior changed)
- **Deferred weather unification**: Would be medium risk, multiple locations to update simultaneously
