# Writing Design Docs

## Our Design Doc Process

Any non-trivial feature or bug fix in the
[Ramble](https://github.com/Ramble-Project/ramble) codebase must start
with a design doc written using this process.

1.  **Directory:** Design docs are stored in their own directory. The directory
    should be named the same as the design doc - e.g.,
    `New_Ramble_Design/New_Ramble_Design.md`. The design doc should start with
    the template provided in the root of the repository. Other supporting files,
    if any, should also be kept alongside in the same directory. With rare
    exceptions, some globally relevant docs may be stored in the root directory,
    but you must check with MAINTAINERS before doing so.

3.  **PR Review:** Once a draft is ready, it's sent for review by creating a
    PR.

4.  **Implementation Guidance:** The approved design doc serves as a crucial
    piece of context when implementing the feature. Once the feature is
    implemented, the feature PR review will include checking it against the
    design doc to ensure completeness. Any skew will indicate that either the
    feature implementation needs to be updated or the design doc needs to be
    updated.

The core benefits of this approach are the collaborative refinement of ideas
and a lightweight process for keeping documentation up-to-date. It also allows
for AI agents to be used continuously throughout the process to faciliate
development where approriate.

