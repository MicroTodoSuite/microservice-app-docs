# Agile Methodology

**Status**: current as of 2026-09-11.

GaCode Solutions manages MicroTodoSuite with Kanban: a continuous flow of work
pulled through a single board, without fixed iterations. This document
describes the team, the board, and the policies that govern the flow.

## Why Kanban

| Factor | Reasoning |
| --- | --- |
| Continuous delivery | Changes ship whenever a pull request is green; fixed iterations would add waiting without adding control. |
| Small team, distinct lanes | Three engineers own separate areas. A shared board with lanes shows each area's flow and the dependencies between them. |
| Externally gated work | Infrastructure applies, quota increases, and approvals wait on people and providers. Kanban makes blocked work visible instead of hiding it inside an iteration. |
| Specification-driven detail | Task-level planning already lives in Spec Kit registers, so the board tracks functional blocks rather than duplicating hundreds of tasks. |

## Team and lanes

| Lane | Owner | Scope |
| --- | --- | --- |
| Infrastructure | Esteban Gaviria (maintainer) | Terraform, AWS and Azure foundations, environment lifecycle, cluster bootstrap |
| CI/CD | Juan Manuel Díaz | Reusable workflows, test suites, supply-chain gates, promotion |
| Observability | Santiago Valencia | Metrics, logs, traces, alerting, and their live evidence |
| Governance | Esteban Gaviria (maintainer) | Constitution, conventions, decisions, and program registers |

## The board

All 2026 work is on one organization project,
[MicroTodoSuite Delivery 2026](https://github.com/orgs/MicroTodoSuite/projects/7).
The two 2025 boards are closed and kept as history.

- **Items are functional blocks.** Each item is an issue in the repository that
  owns the block, and it links to the task register that details it
  (`specs/<feature>/tasks.md`). The register, not the board, is the source of
  truth for task status.
- **Status**: Backlog, In Progress, In Review, Blocked, Done.
- **Lane**: Infrastructure, CI/CD, Observability, Governance.
- **Progress**: the count of ticked tasks in the block's register.

## Flow policies

- **Pull, do not push.** An owner moves a block to In Progress when the lane has
  capacity, not when the block is assigned.
- **WIP limit.** A lane holds at most two blocks In Progress. A third block
  starts only when one of the two moves to In Review, Blocked, or Done.
- **In Review** means the pull requests that complete the block are open.
- **Blocked** means the block waits on something outside the lane — an approval,
  a quota increase, another block — and a comment on the item names it.
- **Done** means every task in the block's register is ticked against a located
  artifact, and every verification task has an observed run, as §7 of the
  conventions requires. A green check alone does not close a block.

## Measures

The board's history supports three measures:

- **Cycle time**: the time from In Progress to Done for a block.
- **Blocked time**: the time a block spends in Blocked, which shows where
  external dependencies slow delivery.
- **Throughput**: the number of blocks and register tasks completed per week.

## What this replaced

The 2025 project used two boards, one for development and one for operations,
with separate columns and WIP limits. The team is now organized by lane rather
than by development and operations, and a single board with a Lane field
replaced the pair.
