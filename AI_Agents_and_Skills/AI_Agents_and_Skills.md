# Optimize and Extend Ramble AI Capabilities
- **Status:** Draft
- **Author:** tteggelit
- **Last Updated:** 2026-10-13

---

## Overview

Today, Ramble includes one top-level `AGENTS.md` file and one skill, the **Workspace
Wizard**. The `AGENTS.md` includes most of the Ramble "ground rules" as well as specific
how-tos for certain things - e.g., Ramble configuration file specifics, Spack information, how to develop new object definitions, etc. The **Workspace Wizard** skill comprehensively describes how to create, interact with, and use Ramble workspaces.


### Current Issues

There are three main areas that should be addressed:

1. The `AGENTS.md` is added to every prompt. Ideally it should only contain the "rules
   of the road" and directions of where to find more specific instructions/how-tos - not
   specifics that aren't globally relevant. For example, there is currently a section
   *How to Write an Application Definition*. Because the `AGENTS.md` is included with
   every prompt, this information is being included whether it is relevant to the task
   at hand and consuming context window and tokens unnecessarily.
1. There are Ramble aspects that could be covered by additional skills: analyzing
   results, debugging experiment issues, interacting with workflow managers, developing
   new Ramble features, etc.
1. For the skills that are there - the Ramble Workspace Wizard and the Writing an
   Application Definition (that is currently embedded in `AGENTS.md`) there are no skill
   evaluators - i.e., `EVAL.txtpb`. For each skill there should be an evaluator to
   ensure that what the skill produces is what's desirable and intended.


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
into new skills or other appropriate locations. New skills will be added under
`.agents/skills` and include at a minimum a `SKILL.md` file and at least one skill
evaluator. Additional files and directories may be needed for certain skills. Examples of
this are:

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
