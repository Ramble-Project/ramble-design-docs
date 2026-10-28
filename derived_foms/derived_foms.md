# Derived Figures of Merit (FOMs)
* **Status:** In-Review
* **Author(s):** Lin Guo
* **Last Updated:** 2026-10-21

---

## Overview

Ramble objects define Figures of Merit (FOMs) using the `figure_of_merit` directive to extract metrics from experiment runs. This design introduces **Derived Figures of Merit**, allowing FOMs to be declaratively computed from other extracted FOMs and experiment variables using either string math/expansion expressions or Python callables.

### Motivation

Currently, `figure_of_merit` supports two extraction mechanisms:
1. **File-based FOMs** (`fom_regex`): Extracted by matching regular expressions against experiment log files.
2. **In-Memory FOMs** (`fom_map_key`): Populated imperatively via `self.add_inmem_fom_value(fom_map_key, value)`.

In practice, many object definitions need to compute secondary or aggregate metrics from values that are already extracted as FOMs or defined as experiment variables. Without native support for derived FOMs, object authors had to override `_prepare_analysis` or `_analyze_experiments` (or register custom analysis phases) to do either or both of the following:

* Manually locate, open, and re-parse log files with custom regular expressions in Python.
* Write out temporary files (e.g., `namd_nspd_stat.out` in `namd`, `stats.out` in `wrf`, `elapsed_seconds` in `py-nemo`) or append lines to existing logs (e.g., `exit-code` appending `Final exit code:` to `{exit_code_log}`, or `nvidia-smi` appending `wc -l` at runtime) just so a file-based `figure_of_merit` regex could read them back. This may be replaced by in-memory FOMs, but boilerplate code is still needed.

Adding declarative derived FOMs eliminates this boilerplate and keeps FOM definitions concise and self-contained.  

### Use Case

1. **Simple Formula Derivation:** An application author defines a file-based FOM `Benchmark days per nanosecond` and wants to report `Nanoseconds per day`. Instead of overriding `_analyze_experiments` and writing a temporary file, the author declares:
   ```python
   figure_of_merit(
       "Nanoseconds per day",
       units="ns/day",
       formula="1.0 / {Benchmark days per nanosecond}",
       fom_type=FomType.THROUGHPUT,
   )
   ```

    To achieve the same without derived FOMs, this is what the current implementation in the builtin repo looks like:

    ```python
    figure_of_merit(
        "Nanoseconds per day",
        log_file=namd_nspd_stat_file,
        fom_regex=r"(?P<ns_per_day>[0-9]+\.*[0-9]*) ns/day",
        group_name="ns_per_day",
        units="ns/day",
    )

    # Override _analyze_experiments to compute ns/day from days/ns
    def _analyze_experiments(self, workspace, app_inst=None):
        """Generate ns/day metric for the experiment"""

        log_path = self.expander.expand_var(self.log_file_str)
        ns_regex = re.compile(self.benchmark_regex)

        dpns = None

        if os.path.isfile(log_path):
            with open(log_path, encoding="utf-8") as f:
                for line in f:
                    match = ns_regex.match(line)
                    if match:
                        dpns = float(match.group("days_per_ns"))

        if dpns:
            nspd = 1.0 / dpns
            nspd_file_path = os.path.join(
                self.expander.expand_var(
                    self.expander.expansion_str(keywords.experiment_run_dir)
                ),
                "namd_nspd_stat.out",
            )
            with open(nspd_file_path, "w+", encoding="utf-8") as f:
                f.write(f"{nspd} ns/day\n")

        super()._analyze_experiments(workspace)
    ```

2. **Context Aggregation via Callable:** A modifier author extracts per-GPU info within a `figure_of_merit_context` and wants a summary FOM in the default (`"null"`) context. The author supplies a callable `formula` that queries `app.get_fom_values(fom_name, context=...)`:
   ```python
   figure_of_merit(
       "GPU Count",
       units="",
       fom_type=FomType.MEASURE,
       formula=lambda app: len(app.get_fom_values("GPU Name", context="gpu")) or None,
   )
   ```

The callable is supported to provide more flexibility and control to Ramble object developers. For instance, for the following derived FOM definition, if there is an FOM named 'a', and there is an experiment variable also named 'a', then the expansion will use the value of the FOM 'a' over the value of the experiment variable 'a'. To use the value of the variable 'a' instead, then the derived FOM can define a callable and explicitly extract the value of the experiment variable 'a' using `app.expander.expand_var_name('a')`.

    ```python
    figure_of_merit(
        "derived_fom_a",
        units="ns/day",
        formula="2 * {a}",
        fom_type=FomType.THROUGHPUT,
   )
    ```

### Requirements

* Extend the `figure_of_merit` directive with a `formula` parameter that accepts either a string expression or a Python callable.
* Provide helper APIs (`get_fom_value` and `get_fom_values`) on Ramble objects to look up resolved FOM values by name and context.
* Support chained derived FOMs (where one derived FOM references another derived FOM) regardless of declaration order.
* Allow derived FOMs to reference resolved FOMs originating from any object in the experiment hierarchy (application, modifiers, workflow manager, etc.) as well as experiment variables.

## Implementation

* Add `formula` support to `figure_of_merit` in `shared_language.py` with clear validation of mutually exclusive arguments.
* Implement iterative derived FOM resolution during experiment analysis in `ApplicationBase.extract_inmem_foms`.
* Expose `get_fom_value` and `get_fom_values` on `ApplicationBase` and `ObjectMixin`.
* Refactor existing builtin applications, modifiers, and workflow managers to use derived FOMs and remove redundant analysis phases and temporary files.
