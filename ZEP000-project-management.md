---
title: Issue Management
authors:
  - @npepinpe
reviewers:
  - @menski
approvers:
  - TBD
creation-date: 2020-11-20
last-updated: 2020-11-26
status: implementable
see-also: []
replaces:
  - "ZEP001-triage-process.md"
superseded-by: []
---

# Summary
[summary]: #summary

Improves upon [ZEP001](/ZEP001-triage-process.md) which is not sufficient under the current circumstances, and specifically aims to increase focus, transparency, and to reduce the size of the backlog in order to make it manageable for the project manager.

# Motivation
[motivation]: #motivation

At the time of writing, there are 466 issues in the Zeebe back log, the oldest being from 2017. Additionally, every quarter, as priorities shift, the back log grows.

With such a large back log, and issues up to 3 years old, it's difficult to say how many still apply and how many are duplicates. It's also very time consuming to review the priorities and ensure they align with the current quarterly goals, or the current product strategy.

This proposal aims to improve the following:

- align the focus of the development team with the quarterly OKRs
- increase transparency for external stakeholders
- reduce the size of the back log
- increase short term developer autonomy
- reduce overhead of the issue management

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Issue management can be viewed as a three step process: categorizing issues, prioritizing issues, and assigning issues. Each step's goal is simplify the next one.

## Categorizing

Categorizing issues is done primarily to help prioritizing issues. By labelling issues appropriately, the product manager can more easily see which issues align with the current OKRs. For example, a bug report scoped to the Go client which affects observability may not aligned with quarterly goals that would focus on performance, whereas a general issue in the broker which is known to affect performance will be. Naturally the project manager will review the issue itself, but the right labels will help speed up the task when it comes to prioritizing the next batch of issues.

An issue is considered categorized on it has a type assigned (e.g. bug, feature, etc.), it is scoped to one or more parts of the project (e.g. broker, gateway, etc.), and its impact on the project assessed (e.g. performance, availability, etc.).

Core team members __should__ categorize issues that they create immediately. Consequently, every week, a team member is tasked with categorizing new issues created by external users.

## Prioritizing

Prioritization is done primarily to ease the task of assigning issues to developers (whether self-assigned or by the project manager).

Issues are prioritized in under four different priorities: critical, high, mid, and low.

Critical issues are fast tracked issues which take precedence over everything, and will interrupt the team's current work, e.g. stop the world issues. For example, a bug in production which has a high chance of causing data loss would be critical and take precedence over any other issue.

High priority issues are those which the team will work on, and will be limited to a small number.

Mid and low priorities are mostly to help the project manager keep track of which issues should become high priority next. Mid priority issues are those which the project manager believe may become high priority in the near future, and low priority those which they believe should be done but most likely will not become high priority anytime soon. Once an issue has been mid priority for some time, it is automatically downgraded to low priority. Once an issue has been low priority for some time, it is automatically closed. Together they form a rolling backlog, which should help speed up the task of backfilling the pool of high priority issues.

## Assigning

The project manager will mark high priority issues as ready to be worked on. 

## Visualization

A [Codetree](codetree.io) board is used to visualize the current workload of the team, with the following columns:

| Backlog | Planned | Ready | In Progress | Needs Review | Done |
|---------|---------|-------|-------------|--------------|------|
|         |         |       |             |              |      |

Issues are attached to different columns based on specific labels. Issues in the `Ready` column are these which team members should work on next. Issues in `In Progress` are these they are currently working on. Issues in `Needs Review` are awaiting a review from a different team member.

## Capacity/limits

The number of issues assigned to a team member will limited to 3, barring some exceptions. The project manager may assign issues directly to developers, or developers may self assign issues directly from the pool of ready issues.

Capacities are set on the priority buckets as well to reflect the project manager's capacity when it comes to prioritization.

## Exceptions

Some issues may not prioritized. These fall broadly into two camps: awaiting information, or questions. 

New issues, especially from external users, may require a team member to gather information before it can be properly categorized and prioritized. These issues are marked as awaiting information.

Questions are issues meant to generate discussion. They might be help requests, or they might be ideas which have no concrete outcome yet. For example, a team member may open a new issue to propose the adoption of LMDB instead of RocksDB, or to propose rewriting the transport layer using gRPC. This issue would be marked as a question, which may be categorized, but will not be prioritized.

In both cases, issues are marked as stale if there is no activity after a month, and automatically closed after another month of inactivity.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Categorizing

Issues are categorized via labels. The goal is to help prioritization by enabling the project manager to quickly filter issues by various dimensions.

### Categories

There are already several labels used to categorize issues, and it's expected that these will change over time as we need them. Broadly speaking, they fall into one of the following kind:

- Type: the kind of issue it is, e.g. bug, enhancement, etc. Type labels are prefixed by `Type:`, e.g. `Type: Bug`, `Type: Enhancement`, etc.
- Impact: how the issue affects the project. For example, `Impact: Availability` may indicate an issue is related to outages, `Impact: Data` one that could cause data corruption, `Impact: Performance` one that could affect performance in a positive or negative way. Additionally these may refer to impacts on the code base itself, e.g. `Impact: Tech Debt` can be used to group issues which should tackle reducing tech debt. 
- Scope: which part of the project the issue affects. For example, `Scope: broker` refers to issues which affect the broker but not the gateway, `Scope: clients/go` to issues which only affect the Go client, etc. An issue may have multiple scopes if it affects more than one of them. For example, a bug in the messaging service would affect both the gateway and the broker.
- Severity: an estimate of how critical the impact is. This category is mostly used for bug reports, and help the project manager prioritize it by informing him of how severe the bug is. For example, a bug report with `Impact: Availability` and `Severity: Critical` may refer to an issue which would deterministically bring about complete outages.

#### Adding a new category

New categories/labels can be created by any team member at their discretion. The author should add a good description to the label so other team members know how/when to use it, and should announce it at the next team meeting so everyone is aware of its existence and any ambiguity can be resolved.

### Responsibility

Every week, a different team member is responsible to look at new issues created by anyone outside of the team, and categorize them. These issues will be labelled with a special label, `Status: Uncategorized`, which the developer can filter on.

## Prioritizing

Issues are prioritized in under four different priorities: critical, high, mid, and low. Normally, only issues which are categorized as `bug`, `feature`, `maintenance`, or `test` will be prioritized.

### Responsibility

It's the project manager's responsibility to prioritize issues, with input from the team. If team members believe an issue in incorrectly prioritized they should reach out to the project manager and discuss it with them. For example, if a developer realizes that there's a low priority issue which, once implemented, would greatly simplify another high priority issue, then they should bring it up with the project manager.

### Critical

Critical issues take precedence over all other issues; they are stop the world issues. For example, a bug which causes data loss to half of the Camunda Cloud customers would be critical. These issues will also typically warrant an immediate patch release.

Critical issues are identified by the label `Priority: Critical`.

### Ready/High priority

High priority issues are those which directly align with our OKRs. They are the only issues which team members work on, with a few exceptions: self assigned issues tackled during slack time, critical issues, etc. For example, if one of our key results is to reach a P3 yield of 99.5%, then a bug report about repeated outages would probably be an high priority issue.

To stay focused, there is only a small pool of 6 unassigned high priority issues. Having a small pool not only makes it clearer to everyone involved what are the immediate next steps, but it makes it fast to reprioritize at the beginning of a new quarter.

High priority issues are identified by the label `Status: Ready`.

### Planned/Mid priority

Mid priority issues are those which are expected to become high priority issues soon. Once the pool of unassigned high priority issue shrinks, then the project manager picks new mid priority issues to fill it.

The count of mid priority issues is capped at 25. This makes it manageable for the project manager to keep track of and know which issues should be added to the high priority pool next.

To help keep the number of mid priority issues low, any issue which stays mid priority for a month will be marked as stale, and after another month, will be downgraded to low priority.

Mid priority issues are identified by the label `Status: Planned`.

### Backlog/Low priority

Low priority issues are those which aren't expected to be prioritized as high any time soon. For example, if the OKRs are focused on stability, an issue to specify the payload of a job using a file in `zbctl` would most likely be low priority.

Low priority issues are not capped. However, to keep the amount manageable, they are marked as stale after a month, and closed automatically after two months.

Low priority issues are identified by the label `Status: Backlog`.

## Assigning

Issues are assigned either directly by the project manager, or team members with no assigned issues can self-assign issues. In both cases, only issues from the ready column may be newly assigned.

There are two exceptions to this: critical issues, and handovers/reassignments. Critical issues, by their nature, will bypass the assignment/capacity limits. Handovers will occur naturally as well for various reasons, and issues must sometimes be reassigned. In this case, the first effort should be to finish the issue via reassignment to another team member. If this is not possible, then the issue should be put back into the ready column.

> For the first iteration, let's try with going slightly over capacity when this occurs. If this doesn't work we can adjust and put them back into the rolling backlog.

## Visualization

A [Codetree](codetree.io) board is used to visualize the current workload of the team, with the following columns:

| Backlog | Planned | Ready | In Progress | Needs Review | Done |
|---------|---------|-------|-------------|--------------|------|
|         |         |       |             |              |      |

Issues are attached to different columns based on specific labels, which is the column name prefixed with `Status:`. 

The first two columns are mostly for the project manager, and represent the rolling backlog mentioned in the prioritization section. The developer workflow essentially start from the column `Ready` and moves rightwards.

### Day-to-day workflow

From the developer point of view, they will generally start with no assigned issues. At this point, they can self assign any of the issues in the ready column, which all have equal priorities. When unsure, they should approach the project manager, who can assign them an issue.

#### In progress

Once they have at least one issue assigned to them, the developer should move it in progress column, and start working on it.

#### Milestones/follow ups

At this point, the project manager may already assign an issue to them as follow up, or they may self-assign one where it makes sense. For example, if they're working on a milestone which contains multiple issue, and the next issue in the milestone is already in the ready column, they can immediately assign it to themselves while working on the first one.

#### Reviews

Once the issue is ready for review, it's moved to the in review column, and the developer can start working on a new issue. However, as it's a natural interruption, it can also be a good point to look into reviews assigned to themselves before starting a new issue.

#### Critical issues

When a critical issue is identified, then the project manager will immediately assign it to one of the developers. At this point, the developer should work on it, but their previous issues remained assigned to themselves.

#### Absence

If the developer will be absent from the project temporarily, they should check in with the project manager on what to do with their assigned issues. If they leave permanently, then their issues will be put back into the rolling backlog.

## Capacity/limits

Each developer should have at most 3 assigned issues at once. The idea here is one possible pre-assigned in the ready column, one in progress, and one in review.

> Note that this is a rule of a thumb, and we should aim to keep the number of assigned issues as low as possible, e.g. 1 in progress. It may also be possible to have more than 3, e.g. critical issues, 2 issues in review, etc.

In order to keep the project manager's work manageable, the mid and low priority buckets will also have capacities. There should be at most 25 mid priority issues, and 50 low priority issues.

If the mid priority overflows, then the project manager must push them down to the low priority issues. If that overflows, they must close some low priority issues.

> Note again that all these numbers are arbitrary. 25 is essentially a single Github issue page, and 50 is simply 2 of them.

## Migration

TODO: how to migrate from the current way of doing things to this proposal

## Exceptions/uncategorized issues

Uncategorized issues fall broadly into two camps: awaiting information, or questions. 

Issues awaiting information are typically issues opened by external users which are incomplete and cannot be categorized and/or prioritized properly. They may also be issues where the project manager has deferred the decision to the product manager/stakeholders. For example, a user may open a bug report about an outage, but without providing which version they are using, how long it lasted, whether or not it recovered, etc., it's quite difficult to prioritize. Or a user may request an interesting feature request, but it's unclear to the project manager if this will be a topic anytime soon.

Questions will typically be issues which are meant to generate discussion. For example, a team member may open a new issue to propose the adoption of LMDB instead of RocksDB, or to propose rewriting the transport layer using gRPC. Once the issue is discussed sufficiently, and a proposal is accepted, then one or more issues would be created instead and go through the normal prioritization process.

In both cases, issues are marked as stale if there is no activity after a month, and automatically closed after another month of inactivity.

# Drawbacks
[drawbacks]: #drawbacks

There are some drawbacks to this approach. 

- It's unclear if it manages to keep a good balance between developer autonomy and product focus. One concern is that it may reduce developer engagement in the product side of things, and lead to them focusing on implementing issues over solving problems.
- It's unclear if the Codetree board will provide us with all the insight that we need - MTTR, lead time, etc.
- It's unclear how it will impact the open source community side of things to start closing issues more aggressively (though the current state is to let them linger...).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TODO

# Prior art
[prior-art]: #prior-art

This proposal is largely inspired by [Kanban](https://en.wikipedia.org/wiki/Kanban_(development), which in my experience has proven great at managing overwhelming backlogs. You can read more about it [here](https://www.digite.com/kanban/what-is-kanban/). The main advantages for us would be that work is pulled in based on capacity, as opposed to being pushed on request. Capacity here doesn't refer to developer hours (though, indirectly, it somewhat does), but rather assigns so called WIP limits for each "step" of the process. In our case we aren't going as far as this, but we do start assigning limits to priorities and to individual backlogs in order to achieve this pull like system.

From a developer point of view, the proposal paves the way for Kanban - the backlog is all high priority issues, and there are clear limits given based on how many assigned issues they have themselves. Lower priorities are there purely to help the project manager keep a handle on which issues can/should go into the backlog soon, and to phase out issues which do not align with the project goals.

# Out of scope
[out-of-scope]: #out-of-scope

No attempt was made to provide an exhaustive list of categories. I expect more categories may come and go over time - the idea here was to give an example of those which exist, and when it makes sense to create one.

Also out of scope is how to handle pull requests from external contributors. These can generate a non trivial amount of work for the team, and I don't have enough experience in the open source field to know what's the best approach to adopt.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What's a good limit for high priority issues?
- What's a good limit for mid priority issues?
- What's a good limit for low priority issues?
- What's a good limit for assigned issues per developer?

# Future possibilities
[future-possibilities]: #future-possibilities



