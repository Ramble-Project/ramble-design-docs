# Optimize and Extend Ramble AI Capabilities
- **Status:** Draft
- **Author:** tteggelit
- **Last Updated:** 2026-10-13

---

## Overview

Today, Ramble includes one top-level `AGENTS.md` file and one skill, the Ramble Workspace
Wizard. The `AGENTS.md` includes most of the Ramble "ground rules" as well as specific
how-tos for certain things. The Ramble Workspace Wizard skill comprehensively describes
how to create, interact with, and use Ramble workspaces.


### Current Issues

There are three main areas that should be addressed:

1. The `AGENTS.md` is added to every prompt. Ideally it should only contain the "rules
   of the road" and directions of where to find more specific instructions/how-tos not
   those kinds of specifics. For example, there is currently a section "How to Write
   an Application Definition". Because the `AGENTS.md` is included with every prompt,
   this information is being included whether it is relevant to the task at hand and
   consuming context window and tokens unnecessarily.
1. There are Ramble aspects that could be covered by additional skills: analyzing
   results, debugging Spack issues, interacting with workflow managers, developing new
   Ramble features, etc.
1. For the skills that are there - the Ramble Workspace Wizard and the Writing an
   Application Definition (that is currently embedded in `AGENTS.md`) there are no skill
   evaluators - i.e., `EVAL.txtpb`. For each skill there should be an evaluator to
   ensure that what the skill produces is what's desirable and intended.


## User Journey


## Goals

* `AGENTS.md` will be refactored and optimized to reduce context impact and provide clear
  guidance of how to find additional information or skills.
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

* Developing or extending third party tools AI capabilities - e.g., developing a new Spack
  application, creating and deploying Cluster Toolkit clusters, etc.


## Design / Architecture

The `AGENTS.md` will remain in place as is and be refactored to extract extraneous
information into new skills or other appropriate locations. New skills will be added under
`.agents/skills` and include at a minimum a `SKILL.md` file and at least one skill
evaluator. Additional files and directories may be needed for certain skills. Examples of
this are:

* `scripts/`: Directory for additional scripts necessary for the skill to function
* `references/`: Directory for additional documentation that is relevant to the skill
* `assets/`: Directory for other assets - e.g., templates - needed by the skill


### `SKILL.md` Format

The `SKILL.md` must follow standard formatting - YAML frontmatter followed by Markdown
content. Skills should be written as prescriptive as necessary to create somewhat
reproducible and predictable steps, but still allow the agent to explore how to achieve
what’s being requested. For example, a skill might dictate that `workspace analyze` be
used to determine experiment status and results to prevent it from reinventing the wheel
to do that, but still leave it free to explore how to debug a failed experiment.


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
