# Declarative Object Deprecation Directive
* **Status:** Draft
* **Author(s):** rfbgo
* **Last Updated:** 2026-11-02

---

## Overview
We need a simple, declarative way to deprecate Ramble object definitions (such as applications or modifiers); or specific versions, workloads, and variants within them before removing them in a future release.

### Current Issue / Current State / Motivation
As Ramble evolves, we occasionally consolidate or retire definitions (for example, replacing `wrfv3` and `wrfv4` with the unified `wrf` application).

Today, there is no standard way to mark an object or feature as deprecated. Removing an object immediately breaks existing user workspaces without warning, while leaving old definitions around indefinitely creates maintenance burden and confusion about which definition to use.

### User Journey / Use Case
* **Definition Author**: Adds a single `deprecate(...)` directive to an object definition (or inside a `with when(...):` block) specifying when it was deprecated, when it will be removed, and what to use instead.
* **End User**:
  * Sees a single, clear warning per CLI command when their workspace uses a deprecated object or feature, telling them how to migrate.
  * Sees deprecation details when inspecting the object with `ramble info`.
* **Maintainer**: When Ramble's version is bumped to the target removal version, automated tests fail until the deprecated object is removed.

### Requirements
* **Declarative**: Fits naturally alongside existing directives like `version`, `variant`, and `conflict`.
* **Flexible**: Supports deprecating an entire object or a conditional subset via `with when(...):`.
* **Clear Messaging**: Supports specifying a `replacement` and/or a custom `msg` with migration instructions.
* **Low Noise**: Emits at most one warning per command (not once per experiment in a matrix) and stays silent during commands like `ramble list`.

## Goals
* Provide a `deprecate(...)` directive usable across all Ramble object types (applications, modifiers, package managers, workflow managers).
* Support conditional deprecation inside `with when(...):` blocks (e.g., deprecating a specific version, workload, or variant).
* Warn users once per CLI command when an experiment uses a deprecated definition, and surface deprecation info in `ramble info`.
* Automatically fail CI tests when the removal version is reached so deprecated objects are cleaned up on schedule.

## Non-Goals
* Deprecating anything outside of object definitions

## Implementation Plan / Detailed Design

### Proposed Directive Syntax
Authors declare `deprecate(...)` at the top level of an object or inside a `with when(...):` block:

```python
# 1. Deprecate an entire object with a replacement
deprecate(
    deprecated_in="0.7.0",
    removed_in="0.8.0",
    replacement="wrf@4.2",
)

# 2. Include a custom migration message
deprecate(
    deprecated_in="0.7.0",
    removed_in="0.8.0",
    replacement="wrf@4.2",
    msg="Update any 'wrfv4' license sections to 'wrf'.",
)

# 3. Deprecate a specific version, workload, or variant
with when("@:3.9.1.1"):
    deprecate(
        deprecated_in="0.7.0",
        removed_in="0.8.0",
        replacement="wrf@4.2",
        msg="WRF 3.x versions are deprecated.",
    )
```

### High-Level Behavior
1. **Shared Language Directive**: Register `deprecate` in `shared_language.py` so it is available to all object types and participates in `with when(...):` conditions just like `conflict`.
2. **Experiment Validation**: Check active deprecations during experiment validation. When an experiment satisfies a deprecation condition:
   * Emit a single user-facing warning for the command.
   * Trigger a deprecation warning for the test suite so `@deprecation.fail_if_not_removed` catches expired deprecations when `ramble_version` is bumped.
3. **`ramble info` Integration**: Display a `deprecations` section in `ramble info <object>` showing the deprecation version, removal version, replacement, and any custom message.

## Alternatives Considered
* **Imperative overrides in each object (`__init__` / `set_variables_and_variants`)**: Requires boilerplate Python methods in every deprecated definition, only works for applications, cannot be scoped with `with when(...):`, and does not show up as a structured attribute in `ramble info`.
* **Immediate removal without deprecation**: Breaks existing user workspaces across minor releases without a transition period.

## Open Questions / Future Work
* Should we add a config option (e.g., `config:fail_on_deprecated:true`) allowing users or CI pipelines to treat deprecation warnings as errors?
* Should `ramble list` visually mark deprecated objects in its table output?

