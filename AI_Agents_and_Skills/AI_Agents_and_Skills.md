# Optimize and Extend Ramble AI Capabilities
- **Status:** In-Review
- **Author:** tteggelit
- **Last Updated:** 2026-10-14

---

## Overview

Today, Ramble includes one top-level `AGENTS.md` file and one skill, the **Workspace
Wizard**. The `AGENTS.md` includes most of the Ramble "ground rules" as well as specific
how-tos for certain things - e.g., Ramble configuration file specifics, Spack information, how to develop new object definitions, etc. The **Workspace Wizard** skill comprehensively describes how to create, interact with, and use Ramble workspaces.


### Current Issues

There are three main areas that should be addressed:

1. The `AGENTS.md` is added to every prompt. Ideally it should only contain the "rules
   of the road" and directions of where to find more specific instructions/how-tos - not
   specifics that aren't globally relevant. For example, there are currently sections
   detailing Spack, how to develop new object defintions, and how to write an application
   definition Because the `AGENTS.md` is included with
   every prompt, this information is being included whether it is relevant to the task
   at hand and consuming context window and tokens unnecessarily.
1. There are Ramble aspects that could be covered by additional skills: analyzing
   results, debugging experiment issues, interacting with workflow managers, developing
   new Ramble features, etc.
1. For the skills that are there - the Ramble Workspace Wizard and the Writing an
   Application Definition (that is currently embedded in `AGENTS.md`) there are no skill
   evaluators - i.e., `evals.json`. For each skill there should be an evaluator to
   ensure that what the skill produces is what's desirable and intended. These evals
   are meant primarily to evaluate that the skill is producing desired, correct, and
   consistent results. Depending on the eval harness used, they can also be used to
   perform model evaluation as well, though this would be a secondary purpose of the
   evals.
   


## User Journeys

### Journey 1: Running, Debugging, and Analyzing a Scaling Study

* **Persona:** An HPC performance engineer using an AI coding assistant (e.g., Gemini
  CLI, Claude Code, or Cursor) to run a GROMACS scaling study on a Slurm cluster.
* **Journey:**
  1. The user asks the agent to set up a GROMACS scaling workspace from 1 to 8 nodes. The 
     agent reads the `AGENTS.md` and loads only the **Workspace Wizard** and **Workflow
     Managers** skills to configure `ramble.yaml`.
  1. After all the experiment jobs have completed, the user asks, *"Did all experiments
     complete successfully?"* The agent dynamically loads the **Job Status** skill and
     determines that 2 of the 8-node jobs failed. The agent then loads the **Failure
     Triage** skill, inspects the experiment logs and Slurm output, and identifies one
     failed due to a node failure and the other ran out of space while writing logs. The
     agent suggests cleaning up storage capacity and the exact `ramble` command to re-run
     only the failed experiments once there is free capacity.
  1. Once all runs succeed, the user asks, *"Analyze the workspace results and provide a
     scaling summary."* The agent loads the **Results Analysis** skill, runs `ramble
     workspace analyze`, generates a parallel efficiency report, and provides a CSV file
     of all FOM results.
* **Outcome:** The user completes the entire experiment lifecycle via natural language,
  while the agent only loads instructions relevant to the current phase — saving tokens
  and providing the data the user needs to complete their task.


### Journey 2: Authoring a New Application Definition

* **Persona:** A domain scientist adding a new benchmark application (`hpcbench`) to
  Ramble.
* **Journey:**
  1. The user prompts the agent to create a new Ramble application definition for
     `hpcbench` using Spack.
  1. The agent reads the `AGENTS.md` and loads the **Definition Author** and
     **Spack Integration** skills.
  1. Following the skill's workflow, the agent checks `ramble software-definitions
     --summary` to avoid package conflicts, writes `application.py` using valid Ramble
     directives (`workload`, `figure_of_merit`, `success_criteria`, etc.), and verifies
     there are no software spec conflicts.
* **Outcome:** The user gets a style-compliant, working `application.py` that can be used
  as-is or further extended by them.


### Journey 3: Developing and Testing a Ramble Code Fix

* **Persona:** A software engineer contributing a bug fix to the Ramble codebase.
* **Journey:**
  1. The user prompts the agent to implement a fix for bug XYZ in `lib/ramble/rable/` and
     create relevant unit tests.
  1. The agent reads the `AGENTS.md` and loads the **Ramble Developer** skill, develops a
     fix for the issue the user described, creates proper Ramble wrapped pytest unit
     tests, runs the tests via `ramble unit-test -k <test_name>`, and validates formatting
     using `ramble style`.
  1. The user asks the agent, *"Update the documentation to reflect the change in behavior
     from the implemented fix."*
  1. The agent loads the **Documentation Author** skill to update the Sphinx `.rst` docs
     properly and then builds the docs locally to ensure they build properly.
* **Outcome:** Contributions follow Ramble's testing and style conventions automatically
  without requiring extensive back-and-forth during PR review.


### Journey 4: Updating and Evaluating a Ramble Skill

* **Persona:** A Ramble maintainer updating a skill after a CLI change.
* **Journey:**
  1. A maintainer updates the `SKILL.md` for the **Workspace Wizard** skill to reflect a
     new `ramble workspace manage` flag.
  1. Before opening a PR, the maintainer runs an evaluation harness against the skill's
     `evals.json` file.
  1. The harness runs the test prompts with and without the updated skill and verifies
     that the agent produces the expected `ramble` commands and configuration outputs.
* **Outcome:** Maintainers have objective, reproducible verification that skill changes
  improve agent accuracy and don't introduce regressions.


## Goals

* `AGENTS.md` will be refactored and optimized to reduce context impact and provide clear
  guidance of where to find additional information or skills. The refactor will focus on
  retaining the "rules of the road" and high level guidance while moving implementation
  and task specifics into appropriate skill files.
* The current set of skills will be expanded to include:
  * Executing Ramble experiments and tracking their status.
  * Analyzing experiment results.
  * Debugging / Diagnosing failed experiments.
  * Using package managers and workload managers as it pertains specifically to Ramble.
  * Writing Ramble documentation.
  * Developing new Ramble features or bug fixes.
* Skill evaluators will be added to allow for using a harness to evaluate skill
  correctness and model performance.
* Ramble is an open source project and all solutions must work for all AI platforms and
  tools - i.e., Google Gemini, Claude, Cursor, Antigravity, etc.
* Ramble can use several different third party tools - package managers, workload
  managers, etc. - and skills should take this into account.


## Non-Goals

* Creating skills that develop or extend third party tools AI capabilities - e.g.,
  developing a new Spack application, creating and deploying Cluster Toolkit clusters, etc.


## Design / Architecture

The `AGENTS.md` will remain in place and be refactored to extract extraneous information
into new skills or other appropriate locations. The `AGENTS.md` would retain/contain the
following information:
* **What is Ramble?** : Basically the Ramble mission and its strengths
* **The Ramble CLI** : Key `ramble` commands, how to disocver other commands and their
  use (i.e., `--help` and `help`), and how to find the documentation in the code base
  (i.e., `docs/command_references.rst`)
* **Ramble Configuration File Overview** : Its YAML format, directory structure, key
  elements, and links to code base documentation (i.e., `docs/workspace_config.rst` and
  `docs/configuration_files.rst`)
* **Package Managers Overview** : Very highlevel description of the package managers and
  links to code base documentation (i.e., `docs/package_managers.rst`)
* **Key Ramble Resources** : Links to examples (`examples/`) and general code base
  documentation (`docs/`)
* **Available Agent Skills** : A table of the skills, their description, and a link to
  their location

Links to the code base and documentation should be relative to the repo root (since that
is where the `AGENTS.md` lives). This will ensure the agent is using whatever the latest
version is of these resources. If documentation links instead point to the ReadTheDocs
pages, it would create unnecessary permission prompts for the agent to access that URL
(unless the use has specifically allowed that in their agent configuration) and may point
to out of date documentation for the running code base.

New skills will be added under `.agents/skills` and include at a minimum a `SKILL.md` file
and at least one skill evaluator. Additional files and directories may be needed for
certain skills. Examples of this are:

* `scripts/`: Directory for additional scripts necessary for the skill to function
* `references/`: Directory for additional documentation that is relevant to the skill
* `assets/`: Directory for other assets - e.g., templates - needed by the skill


### `SKILL.md` Format

The `SKILL.md` must follow standard formatting (e.g.,
[Agent Skills Standard](https://agentskills.io/specification)) - YAML frontmatter followed
by Markdown content. Skills should be written as prescriptive as possible to facilitate
reproducible and predictable steps. Scripts should be written that the skill can utilize to
help with reproduciblity. For example, a skill might dictate that `ramble workspace analyze`
be used to determine experiment status and results while using a script to triage common
failure cases (e.g, full fileystems, OOM, etc.). This approach will facilitate
reproducibility and discourage rabbit holes like using `objdump`, `nm`, or `strings` for
discovery. When developing new skills, if you notice the agent constantly writing custom
Python scripts or using tools like previously mentioned, that's an indicator a skill script
or other guardrails should be investigated.

Likewise, skills should be narrowly focused allowing the agent to compose them together
to achieve more complex tasks. For example, rather than write one skill that analyzes
the Ramble experiment results, checking if any experiment jobs are still pending or
running, and triaging any failed experiments - it's better to split those into three
separate skills - a results analyzer, a job status checker, and a failure triage skill.
That will reduce the need for the skill to have complex if/then logic and complex steps.

A propsed list of initial skills is:

| Skill | Description |
| :--- | :--- |
| **Workspace Wizard** | Interactive setup of experiment workspaces, scaling matrices, and YAML configurations |
| **Definition Author** | Authoring Ramble object definitions (e.g., Applications, Modifiers, Package/Workflow Managers, etc.) |
| **Experiment Runner** | Using Ramble to perform experiments - execution lifecycle (`ramble on`), dry-run validation, and execution debugging |
| **Job Status** | Experiment execution matrix, detecting active vs completed jobs |
| **Failure Triage** | Root cause diagnosis for confirmed failed runs (NODE_FAIL, timeouts, OOM, spool aborts, logs) |
| **Results Analyzer** | FOM extraction and speedup & scaling efficiency metrics |
| **Benchmark Report** | Comprehensive HPC benchmark reports comparing multiple Ramble workspaces across applications, workloads, and platforms |
| **Documentation Author** | Writing Sphinx/reST documentation in `docs/`, building HTML docs, and link checking |
| **Spack Integration** | Spack specs (`pkg@ver %compiler`), environment mapping, and concretization troubleshooting |
| **Workflow Managers** | Slurm batch directives (`#SBATCH`), partition settings, and launcher overrides |
| **Developer Guide** | Guidelines for codebase contributors: pytest fixtures (`make_workspace_from_config`), directive lazy loading, and style checks |


#### YAML Frontmatter

At a minimum, the following should be included:

* `name` : Max 64 characters. Lowercase letters, numbers, and hyphens only. Must not start
  or end with a hyphen.
* `description` : Max 1024 characters. Non-empty. Describes what the skill does and when
  to use it.
* `license` : License name or reference to a bundled license file. An example valid
  frontmatter looks like

```
---
name: example-skill
description: An example skill.
license: Apache-2.0 or MIT
---
```


### Evaluator Format

Evaluators should be written for every skill to ensure the skill is producing usable and
correct outputs. Evaluators will use the
[`evals.json`](https://agentskills.io/skill-creation/evaluating-skills) format.


## Alternatives Considered

* Maintain status quo


## Open Questions / Future Work

### Evaluator Harnesses

While it's outside the scope of this design doc, possible evaluator tools/harnesses are:

* [agent-skills-eval](https://github.com/darkrishabh/agent-skills-eval) : A/B evaluation
  (with/without skill) and uses the `evals.json` format
* [skill-eval-harness](https://github.com/adewale/skill-eval-harness) : Similar to above
  but uses a different format for the eval file
* Anthropic’s
  [skill-creator skill](https://github.com/anthropics/skills/tree/main/skills/skill-creator) :
  Part of the skill is to evaluate the skill based on the `evals.json` format
* AWS sample [agent-skill-eval](https://github.com/aws-samples/sample-agent-skill-eval) :
  Yet another tool based on the `evals.json` format
* [Harbor](https://github.com/harbor-framework/harbor) : Containerized solution more
  geared towards model evaluation
* NVIDIA’s [SkillEvaluator](https://docs.nvidia.com/skills/skillevaluator/) : 3-tier evaluation tool that uses the `evals.json` format and runs containerized tests via Harbor
* [Promptfoo](https://github.com/promptfoo/promptfoo) : General LLM/agent evaluation tool and GitHub Action that uses its own YAML format
* [skills-evals](https://github.com/ahnafyy/skills-evals) : Lightweight CI regression and trigger evaluation tool that uses the `evals.json` format
* Addy Osmani’s [agent-skills](https://github.com/addyosmani/agent-skills) : Reference 3-tier CI evaluation scripts based on the `evals.json` format

| Tool | Description | Pros | Cons |
|---|---|---|---|
| [agent-skills-eval](https://github.com/darkrishabh/agent-skills-eval) | A/B evaluation (with/without skill) and uses the `evals.json` format. | - Natively uses `evals.json` and validates `SKILL.md` frontmatter.<br>- Works across AI platforms and tools via OpenAI-compatible APIs (Gemini, Claude, OpenAI).<br>- `CI=true` option for CI friendly running, path filtering (`--include`/`--exclude`), parallel runs, assert what tools were called, and JSON/HTML reports. | - Only evaluates via API calls rather than running a CLI agent.<br>- Requires a Node.js runtime (`npx`/npm). |
 [skill-eval-harness](https://github.com/adewale/skill-eval-harness) | Similar to above but uses a different format for the eval file. | - Python tool (`uv`/`pip`) with adapters for multiple agent CLIs (Claude, Codex, Pi, Vibe) and an offline stub for CI testing.<br>- Can grade based on code rules or an LLM judge, flags cases that either succeed without skill or skill adds no value, and checks for eval prompts that may give away how the skill should proceed (a.k.a., "eval leakage"). | - Uses `shared-benchmark.json` instead of `evals.json`, requiring a format conversion step in CI.<br>- Requires writing new Python adapter for `agy` CLI. |
| Anthropic’s [skill-creator skill](https://github.com/anthropics/skills/tree/main/skills/skill-creator) | Suite of skill development/refinement tools that includes evaluation based on the `evals.json` format. | - Validates, evaluates (using `evals.json`), and benchmarks skills.<br>- Includes scripts to help tune `SKILL.md` descriptions. | - Coupled to Claude CLI and does not support other AI platforms (e.g., Gemini, Cursor, Antigravity).<br>- Designed for interactive human review rather than automated CI runs. |
| AWS sample [agent-skill-eval](https://github.com/aws-samples/sample-agent-skill-eval) | Yet another tool based on the `evals.json` format. | - Python CLI (`pip`) that uses `evals.json`.<br>- Includes a fast static checker that scans `SKILL.md` and `scripts/` for unsafe commands or secrets without needing LLM API keys.<br>- Combines static audit, A/B evaluation, and trigger accuracy into a single score. | - Maintained as an `aws-samples` repository rather than a supported package, so it may need to be pinned or vendored. |
| [Harbor](https://github.com/harbor-framework/harbor) | Containerized solution more geared towards model evaluation. | - Runs full coding agents inside isolated containers where Ramble, Spack, and workload managers can execute real commands and be verified via test scripts.<br>- Supports multiple agent harnesses and models. | - Does not use `evals.json` natively (uses `task.toml`, `instruction.md`, and `Dockerfile`).<br>- High container build and runtime overhead for standard PR checks. |
| NVIDIA’s [SkillEvaluator](https://docs.nvidia.com/skills/skillevaluator/) | 3-tier evaluation tool that uses the `evals.json` format and runs containerized tests via Harbor. | - Natively uses `evals.json` and automatically compiles test cases into Harbor container tasks.<br>- Supports a 3-tier CI workflow: frontmatter/schema validation (Tier 1), trigger/deduplication checks (Tier 2), and containerized A/B evaluation across multiple agents (Tier 3). | - Tier 3 execution requires Harbor infrastructure and agent CLI credentials, making it heavy for per-commit CI runs. |
| [Promptfoo](https://github.com/promptfoo/promptfoo) | General LLM/agent evaluation tool and GitHub Action that uses its own YAML format. | - Mature CI integration with an official GitHub Action that posts PR results and enforces pass/fail thresholds.<br>- Supports all major AI platforms (Gemini, Claude, OpenAI/Codex).<br>- Evaluates both positive and negative skill triggering. | - Uses `promptfooconfig.yaml` instead of `evals.json`, requiring a wrapper script to load `evals.json`. |
| [skills-evals](https://github.com/ahnafyy/skills-evals) | Lightweight CI regression and trigger evaluation tool that uses the `evals.json` format. | - Zero-dependency CLI designed specifically to catch regressions in `AGENTS.md` and `SKILL.md` during CI runs.<br>- Evaluates both positive and negative skill triggering . | - Focuses on trigger routing and prompt responses rather than executing multi-step CLI workflows. |
| Addy Osmani’s [agent-skills](https://github.com/addyosmani/agent-skills) | 3-tier CI evaluation scripts based on the `evals.json` format. | - Fast, zero-cost frontmatter validation and cross-skill trigger collision checks.<br>- Supports isolated and ephemeral sandboxed test environments. | - Provided as reference repository scripts rather than an installable package, so the runner scripts would need to be adapted and maintained in Ramble. |