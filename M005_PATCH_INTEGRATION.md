# GACE M005: MultiMesh redundant upload

## Scope and cause

In the tested 10,000-instance `NATIVE_MIXED` workload, individual initialization creates a CPU-side MultiMesh data cache. Each later full-buffer submission uploads the complete buffer directly, then marks the cached regions dirty for deferred processing. Unmodified Godot uploads the same complete data a second time when it processes those regions.

The GACE patch records a successful direct upload and omits only that redundant deferred GPU upload. It still copies the new data into the CPU cache, maintains dirty-region history, and processes automatic AABB updates. A later individual edit clears the skip flag so normal dirty upload occurs. The flag is reset on allocation, motion-vector activation, and completion of dirty processing. Motion-vector and physics-interpolation paths retain the original upload behavior.

The patch is confined to `D:\GACE\GACE-Engine\servers\rendering\renderer_rd\storage_rd\mesh_storage.cpp` and `mesh_storage.h`. It adds no public API. Optional counters and logging in these files are compiled only with `GACE_MULTIMESH_DIAGNOSTICS`; the optimization does not depend on them.

## Build

Godot SCons already supports `cppdefines`; no new build option is needed. From the engine root, with MSVC and the Windows SDK available:

```powershell
scons platform=windows target=editor arch=x86_64 optimize=speed accesskit=no d3d12=no cppdefines=GACE_MULTIMESH_SKIP_REDUNDANT_UPDATE extra_suffix=gace_mm_prod -j8
```

This build leaves diagnostics off. The `accesskit=no` and `d3d12=no` flags match the dependencies available on this host; Vulkan remains enabled. For an explicitly requested diagnostic build, add `GACE_MULTIMESH_DIAGNOSTICS` to `cppdefines` and use a distinct `extra_suffix`. Do not use that diagnostic build for ordinary release measurements.

Validation artifacts referenced below are under `D:\GACE Projects\gace-benchmarks\.gace-runs\`.

## Validation record

- Measured 10,000-instance `NATIVE_MIXED`: mean observed frame time 0.5420 ms reference and 0.4963 ms optimized, 8.43% lower for this workload. This is not an engine-wide improvement claim.
- State-transition regression: individual/full updates in both orders and in the same frame, repeated full updates, sparse 512-instance region boundaries, custom/automatic AABB, and independent MultiMesh resources passed expected-transform and pixel comparisons. See `.gace-runs/m005_t06_regression/`.
- Deterministic TAA and physics-interpolation fallback comparisons each had zero changed pixels at two timestamps on Vulkan Forward+ / RTX 4060. See `.gace-runs/m005_t071_taa_opt/deterministic.json` and `.gace-runs/m005_t071_physics_opt_v2/deterministic.json`.
- Diagnostic counters showed direct and deferred full uploads in fallback paths, while the ordinary optimized mixed path had zero deferred full uploads during capture. Diagnostics are evidence of the path taken, not a substitute for rendered-image validation.
- The original and optimized `NATIVE_INDIVIDUAL` and `NATIVE_BULK_ONLY` benchmark results showed no material regression. The complete performance matrix need not be repeated for a build that uses the same algorithm.

## Prepared production build (2026-10-22)

The build command above completed with exit code 0 using MSVC 14.3 and Windows SDK 10.0.26100.0. The executable is `D:\GACE\GACE-Engine\bin\godot.windows.editor.x86_64.gace_mm_prod.console.exe`. A first configuration check without `accesskit=no d3d12=no` stopped before compilation because those optional dependencies are unavailable on this host. The completed build reported a nonfatal optional ANGLE dependency warning.

- Headless `res://native/validate_m005.gd`: exit code 0; all three native update strategies passed at 1,000 instances and three timestamps; CSV check passed. Log: `.gace-runs/m005_t08_prod_headless/`.
- Graphical `NATIVE_MIXED` validation: runner PASS, Godot exit code 0, 10,000 instances, Vulkan Forward+, RTX 4060, 1152 x 648. Both fixed-time images matched the existing reference pixel for pixel (zero changed pixels of 746,496 each); motion between timestamps changed 46,520 pixels. No shader-cache error or diagnostic log output appeared. Result: `.gace-runs/m005_t08_prod_validate_f734fc23/`.
- Both runs reported `Failed to read the root certificate store` in stderr. This did not affect the reported rendering comparison. No Godot process remained after validation.

This production build omits the optional diagnostic define and keeps the validated optimization algorithm. The graphical check confirms the prepared build's rendered output; it is not a new performance measurement.

## Limits and Git checkpoint

The performance result covers the specified workload, renderer, GPU, viewport, and build settings. Frustum inclusion does not prove every instance is GPU-visible. CPU/GPU render monitors and observed frame intervals measure different quantities. Preserve historical benchmark outputs under `.gace-runs/`.

For the engine checkpoint, review only the two modified MeshStorage files. Keep `GACE_MULTIMESH_DIAGNOSTICS` off in the ordinary production build. No commit, push, branch change, or reset is part of this preparation.
