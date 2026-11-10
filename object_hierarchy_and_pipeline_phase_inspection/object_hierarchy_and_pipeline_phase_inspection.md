# Object Hierarchy and Pipeline Phase Inspection
* **Status:** Draft
* **Author(s):** chinglung880218
* **Last Updated:** 2026-11-09

---

## Overview
Ramble objects (applications, modifiers, package managers, and workflow managers) rely on multi-level class inheritance to share common functionality. Furthermore, their execution lifecycles are composed dynamically: classes in the inheritance chain as well as attached plugins contribute pipeline phases that Ramble merges into a directed acyclic graph (`PhaseGraph`). Currently, users lack a straightforward CLI mechanism to inspect the inheritance hierarchy, namespaces, and source file paths of an object, or to explore the topologically sorted pipeline phases contributed by one or more objects.

This design proposes introducing a new `ramble inspect` command to display:
1. The class inheritance hierarchy, namespaces, and source file paths used to construct an object.
2. The topologically ordered pipeline phases for an object, including phase ordering constraints and defining classes.
3. **(Stretch Goal)** Aggregated pipeline phases across multiple objects or targeted experiments within an active workspace.

## Motivation
Today, `ramble info <object>` is the primary CLI command for querying an object's metadata (such as workloads, FOMs, and tags). However, it falls short of providing structural and execution insight in three critical areas:

1. **Hidden Inheritance Hierarchy:**
   In Ramble, many objects are lightweight and inherit most of their logic from upstream base classes. For example, `applications/hpl/application.py` is only 44 lines long, with its workloads, FOMs, and custom phases defined upstream.

   However, running `ramble info hpl` completely hides this hierarchy, showing only:
   ```text
   $ ramble info hpl
   Application: hpl
   ```
   Users cannot see parent classes in the inheritance chain (where `->` indicates "inherits from"):
   ```text
   hpl -> HplBase -> ExecutableApplication -> ApplicationBase -> ObjectMixin
   ```
   Without this visibility, users trying to understand or modify an application must manually grep the repository to find where directives and phases were defined.

2. **Unsorted and Incomplete Phase Information:**
   Running `ramble info --attributes registered_phases hpl` prints the raw dictionary insertion order of `obj.phase_definitions` rather than walking the topological `PhaseGraph`:
   ```text
   setup:
       bootstrap_utilities  get_inputs  license_includes  make_experiments  write_status  calculate_values
   ```
   Here, `calculate_values` is misleadingly printed at the very end (after `make_experiments` and `write_status`), even though it is registered with `run_before=["make_experiments"]`. Furthermore, users cannot see which class in the inheritance chain defined each phase or what ordering constraints (`run_before` / `run_after`) apply.

3. **No Multi-Object or Workspace Experiment Phase Aggregation:**
   In actual workspace execution, an experiment's `PhaseGraph` merges phases from the Application, Package Manager (e.g., `spack`), and attached Modifiers. `ramble info` operates strictly on a single object definition and provides no mechanism to preview this combined phase execution order without running the pipeline.

### Use Case
* **Plugin Developers & Maintainers:** A developer creating a new HPL variant or modifier wants to inspect which base classes `hpl` inherits from, locate their source files on disk, and verify where a custom `register_phase` hook will be inserted into the `setup` or `analyze` pipeline.
* **End Users Debugging Workspaces:** A user wants to understand the exact sequence of `setup` or `analyze` phases that will execute for a specific combination of application, package manager, and modifiers (or for a targeted experiment in their workspace).

## Goals
* **Phase 1 (Object Hierarchy):** Allow users to inspect the inheritance hierarchy, full module namespaces, and file paths for any Ramble object.
* **Phase 2 (Single-Object Pipeline Phases):** Allow users to explore topologically sorted pipeline phases, ordering constraints, and origin classes for a single object.
* **Phase 3 (Stretch Goal - Multi-Object / Workspace Experiment Phases):** Allow users to view the combined `PhaseGraph` across multiple specified objects or for targeted experiments in a workspace.


## Implementation Plan

### Phase 1: Object Inheritance Hierarchy Inspection
Every instantiated Ramble object inherits from `ObjectMixin` and has a Python Method Resolution Order (`obj.__class__.__mro__`). Each class in the MRO has a `__module__` corresponding to its Ramble namespace (e.g., `ramble.app.builtin.hpl`, `ramble.base_app.builtin.hpl`, `ramble.base_cls.builtin.application-base`).

Using `importlib.util.find_spec(cls.__module__).origin` (similar to `ramble.repository.list_object_files`), we resolve the exact source file path for every class in the inheritance chain.

**Proposed CLI Examples:**
```bash
# Default behavior (omitting mode flag defaults to --hierarchy):
$ ramble inspect hpl

# Explicit flag or shorthand:
$ ramble inspect --hierarchy hpl
$ ramble inspect -H hpl
```

**Proposed Output Example:**
```text
$ ramble inspect --hierarchy hpl

Object Hierarchy for: hpl (application)
--------------------------------------------------------------------------------
Hpl [ramble.app.builtin.hpl]
  path: .../var/ramble/repos/builtin/applications/hpl/application.py
  └──► HplBase [ramble.base_app.builtin.hpl]
         path: .../var/ramble/repos/builtin/base_applications/hpl/base_application.py
         └──► ExecutableApplication [ramble.base_cls.builtin.executable-application]
                path: .../var/ramble/repos/builtin/base_classes/executable-application/base_class.py
                └──► ApplicationBase [ramble.base_cls.builtin.application-base]
                       path: .../var/ramble/repos/builtin/base_classes/application-base/base_class.py
                       └──► ObjectMixin [ramble.base_cls.builtin.object-mixin]
                              path: .../var/ramble/repos/builtin/base_classes/object-mixin/base_class.py
```

### Phase 2: Single-Object Pipeline Phase Exploration
Instead of iterating over raw keys in `obj.phase_definitions[pipeline]`, phase inspection will construct a `ramble.graphs.PhaseGraph(obj.phase_definitions[pipeline], obj)` and walk the graph. Because a DAG with independent branches admits multiple valid topological sorts, the displayed list represents the specific valid topological execution order resolved by Ramble's `graphlib.TopologicalSorter`.
In verbose mode (`-v`), each phase node will also display:
* The class in the MRO where `_<phase_name>` is implemented.
* Ordering dependencies (`node._order_after` and `node._order_before`) and `when` conditions.

**Proposed CLI Examples:**
```bash
# Basic usage (prints DAG flow and topological order for all registered pipelines):
$ ramble inspect --phases hpl

# Filter to a specific pipeline:
$ ramble inspect --phases --pipeline setup hpl

# Verbose mode (displays defining classes and ordering constraints):
$ ramble inspect --phases --pipeline setup -v hpl
```

**Proposed Output Example (Default Mode):**
```text
$ ramble inspect --phases --pipeline setup hpl

Execution Flow (DAG):
--------------------------------------------------------------------------------
• license_includes (independent)

• bootstrap_utilities
  └──► get_inputs
         └──► make_experiments
                └──► write_status

• calculate_values
  └──► make_experiments (merged above)

Topological Execution Order:
--------------------------------------------------------------------------------
  1. bootstrap_utilities
  2. license_includes
  3. calculate_values
  4. get_inputs
  5. make_experiments
  6. write_status
```

In verbose mode (`-v`), the topological execution order expands to display defining classes and ordering constraints:
```text
$ ramble inspect --phases --pipeline setup -v hpl

Phase Details (Resolved Topological Execution Order):
--------------------------------------------------------------------------------
*(Note: Represents one valid topological execution order resolved by graphlib.TopologicalSorter)*
  1. bootstrap_utilities
       defined in: ApplicationBase [ramble.base_cls.builtin.application-base]
       run_before: ['get_inputs']
  2. license_includes
       defined in: ApplicationBase [ramble.base_cls.builtin.application-base]
  3. calculate_values
       defined in: HplBase [ramble.base_app.builtin.hpl]
       run_before: ['make_experiments']
  4. get_inputs
       defined in: ApplicationBase [ramble.base_cls.builtin.application-base]
  5. make_experiments
       defined in: ApplicationBase [ramble.base_cls.builtin.application-base]
       run_after:  ['get_inputs', 'software_create_env']
  6. write_status
       defined in: ApplicationBase [ramble.base_cls.builtin.application-base]
       run_after:  ['make_experiments']
```

### Phase 3 (Stretch Goal): Aggregated / Workspace Experiment Phase Inspection
To inspect the combined phases of multiple objects or a workspace experiment:
* **Workspace Mode (`ramble -w <ws> inspect --phases [--where ...]`):**
  Uses `workspace.build_experiment_set()` and `app_inst.build_phase_order()` to walk `app_inst._pipeline_graphs[pipeline]` for each filtered experiment, printing each phase along with its owning object (`node.obj_inst.scoped_name`).
* **Multi-Object Aggregation (`ramble inspect --phases hpl --package-manager spack --modifier lscpu`):**
  Instantiates the requested objects, merges their `all_pipeline_phases(pipeline)` nodes and edges into a single `PhaseGraph`, and resolves a valid topological execution order across all contributed phases.

**Proposed CLI Examples:**
```bash
# Workspace mode (inspects resolved experiment DAG using global -w):
$ ramble -w my_workspace inspect --phases

# Ad-hoc multi-object aggregation:
$ ramble inspect --phases hpl --package-manager spack --modifier lscpu
```

**Proposed Output Example (Multi-Object Aggregation):**
```text
$ ramble inspect --phases --pipeline setup hpl --package-manager spack --modifier lscpu

=== Aggregated Execution Flow (DAG): setup (hpl + spack + lscpu) ===
--------------------------------------------------------------------------------
• [app:hpl] license_includes (independent)

• [app:hpl] bootstrap_utilities
  └──► [app:hpl] get_inputs
         └──► [app:hpl] make_experiments
                └──► [app:hpl] write_status

• [app:hpl] calculate_values
  └──► [app:hpl] make_experiments (merged above)

• [pkg:spack] software_create_env
  └──► [pkg:spack] deploy_artifacts
         └──► [pkg:spack] define_package_paths
                └──► [app:hpl] make_experiments (merged above)

Phase Details (Resolved Topological Execution Order):
--------------------------------------------------------------------------------
*(Note: Represents one valid topological execution order resolved by graphlib.TopologicalSorter)*
   1. [application:hpl       ] bootstrap_utilities
   2. [application:hpl       ] license_includes
   3. [application:hpl       ] calculate_values
   4. [package_manager:spack ] software_install_requested_compilers
   5. [application:hpl       ] get_inputs
   6. [package_manager:spack ] software_create_env
   7. [package_manager:spack ] software_configure
   8. [package_manager:spack ] software_mirror_configs
   9. [package_manager:spack ] software_install
  10. [package_manager:spack ] deploy_artifacts
  11. [package_manager:spack ] define_package_paths
  12. [application:hpl       ] make_experiments
  13. [application:hpl       ] write_status
```

### Implementation Details & File Modifications
1. **New CLI Command (`lib/ramble/ramble/cmd/inspect.py`):**
   * Implement `setup_parser(subparser)` with arguments:
     * `object`: Primary object spec to inspect.
     * `--hierarchy / -H`: Flag to display the inheritance chain (default when mode flag is omitted).
     * `--phases / -p`: Flag to inspect pipeline phases and DAG execution flow.
     * `--pipeline`: Filter to specific pipeline (default: all registered pipelines).
     * `--package-manager`, `--modifier`: For Phase 3 multi-object aggregation.
     * `-v / --verbose`: Verbose mode showing phase defining classes and constraints.
   * Implement `inspect(parser, args)` routing to hierarchy and phase formatters.

2. **Core Helper Methods (`var/ramble/repos/builtin/base_classes/object-mixin/base_class.py`):**
   * Add `inheritance_chain(self)` returning a list of dicts with `name`, `alias`, `namespace`, and resolved `file_path` for each class in `self.__class__.__mro__`.
   * Add `phase_origin_class(self, phase_name)` helper to inspect `self.__class__.__mro__` and identify the specific class implementing `_{phase_name}`.

3. **Phase Graph & DAG Rendering (`lib/ramble/ramble/cmd/inspect.py`):**
   * Construct `ramble.graphs.PhaseGraph` for the requested object(s).
   * Walk `graph.adj_list` to render the clean dependency tree (`Execution Flow (DAG)`).
   * Iterate over `graph.walk()` to print the resolved topological order and metadata.

4. **Testing Strategy (`lib/ramble/ramble/test/cmd/inspect.py`):**
   * Add unit tests using `RambleCommand("inspect")` and `mutable_mock_apps_repo`:
     * Test `--hierarchy` output on mock applications with base templates.
     * Test `--phases` ensures topologically valid ordering (e.g., verifying `calculate_values` precedes `make_experiments`).
     * Test multi-object phase aggregation combining an application with mock modifiers/package managers.

5. **Edge Cases & Design Considerations:**
   * **Handling Multiple Inheritance:** If an object inherits from multiple mixins or base classes, the displayed tree follows Python's linearized MRO, annotating secondary mixins clearly with a `[mixin]` tag.
   * **Evaluating `when=[...]` Phase Constraints:** When inspecting a standalone object, phases are filtered against the object's default variants (`obj.default_variants`). In workspace mode, phases are dynamically evaluated against the experiment's resolved variant set.
   * **Pipeline Selection Behavior:** If `--pipeline` is omitted, `ramble inspect --phases` iterates through all registered pipelines containing phases (e.g., `setup`, `analyze`), rendering each under its respective section header. Users can supply `--pipeline <name>` to isolate a specific pipeline.

## Open Questions / Future Work
1. For the stretch goal, should we prioritize workspace experiment inspection (`-w <ws> --where ...`) or ad-hoc multi-object CLI aggregation first?
2. In the future, could `--phases` also support an optional `--format mermaid` or `--format dot` to export graphical diagrams for documentation?
3. Should `ramble info`'s display of `registered_phases` eventually be updated to follow topological order, or should it deliberately remain a dump of raw declared dictionary keys, leaving all graph-aware execution ordering to `ramble inspect`?
