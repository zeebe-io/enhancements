---
title: Zeebe LFS Add On
authors:
  - pihme
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: 2020-07-13
last-updated: 2020-07-13
status: provisional
---

# Summary
[summary]: #summary

This ZEP proposes an add on that outsources the storage of large data into an external system (e.g. Google Cloud Storage, AWS S3). Zeebe is relieved of handling large data and can deliver better performance. 

# Motivation
[motivation]: #motivation

Zeebe is not ideal for handling or storing large amounts of data. Customers sometimes want to use large amounts of data within a workflow.

The large data could be offloaded to an external system and Zeebe handles only a tiny reference to the data. 

This tiny reference will be small enough that Zeebe can be configured to use small message sizes to achieve better performance. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are at least two ways this could be implemented:

## I) Implementation as part of the Gateway

*Idea* 
* Gateway will inspect data stream for variables that exceed a certain length
* Such variables will be stored in an external storage and Zeebe will replace them internally with a reference
* When variables are requested by a client the reference is resolved and the content returned 
* When exporting the reference is resolved and the content returned

*Pro*
* Fully transparent to the clients
* Could be offered as an added value feature to paying customers
* Data can be evicted from external storage when log entries are deleted

*Contra*
* We break the design constraint to be independent of external systems.
* Gateway bandwidth still hit with large data
* Probably not scalable to very large data (> 1GB)
* Problem of large data is only moved along to the exporters



## II) Implementation as part of the Client API

*Idea* 
* Client API is extended to allow setting large variables
* These are then streamed to external storage on the client side, and the client puts in the replacing identifier or URL as the variable content
* Gateway, Zeebe, exporters only see the reference to the object in external storage, never the real data

*Pro*
* Could be interesting for users in terms of security and privacy (Zeebe will handle a reference to the data, but never see the actual data; all authorization and authentication is handled on client side)
* More flexible/expandable if users have strong preference on which storage to use
* Can support large data of arbitrary size
* Gateway/exporter bandwidth never sees the large data

*Contra*
* Needs to be implemented for each client API we support
* No full audit trail, because external system is out of our control. Maybe we cannot even access it
* Workers need to be aware that variable content might be just a reference to data stored elsewhere. This is something we could add as part of the client API, but then again we have to do it for every client we support.
* Susceptible to user error (e.g. if a developer sets the big variable directly, and doesn't use the API for setting large variables, then Zeebe might choke on it)
* Unclear, how data in external storage can be evicted (Zeebe can give hints, e.g. after a workflow instance is closed, but somehow has to respect those hints)

<!--
Explain the proposal as if it was already included in the product and you were teaching it to another user/contributor. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how our users/contributors should *think* about the feature, and how it should impact the way they use our product. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing users/contributors and new users/contributors.

For user facing ZEPs, this section should describe the benefits or changes the users will experience, from the point of view of the user.

For maintenance/non-user facing ZEPs, this section should focus on how other contributors should reason about the changes, and give concrete examples of its impact, both short term and long term.

For organizational ZEPs, this section should provide an example-driven introduction to the new policy or process, and explain its impact on the development process in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
Will be filled if the guide-level explanation has merit.

<!--
This is the technical portion of the ZEP. After reading it, a contributor should understand/know the following:

- [ ] The impact of the changes on other features is clear.
- [ ] The implementation is delineated
- [ ] Known corner cases are listed and addressed

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.


- [ ] Does the ZEP affect the official Zeebe distribution (e.g. configuration, logging)?
- [ ] Does the ZEP require coordination with the platform team?
- [ ] Does the ZEP require coordination with the Operate team?


## Compatibility

This section should also list incompatible changes of Zeebe's public APIs, and make it explicit should there be any breaking changes.

Should there be any breaking changes, it should explicitly describe the migration path. Should there be no possible migration paths, it should instead explain why it is not possible, and why we decided that the benefits are worth breaking compatibility.

After reading this section, a contributor should know the following:

- [ ] Will it be possible to upgrade a Zeebe cluster?
- [ ] If applicable, what is the upgrade procedure? Is it automated?
- [ ] Does the ZEP break compatibility in the Go client?
- [ ] Does the ZEP break compatibility in `zeebe-client`?
- [ ] Does the ZEP break compatibility in `zeebe-bpmn-model`?
- [ ] Does the ZEP break compatibility in `zeebe-exporter-api`?
- [ ] Does the ZEP break compatibility in `zeebe-protocol`?
- [ ] Does the ZEP break compatibility in `zeebe-gateway-protocol`?
- [ ] Does the ZEP break compatibility in `zeebe-test`?

## Testing

You should describe what is the overall functionality that should be tested.

If you are omitting tests, explain why, and explain the impact if it fails, specifically the worst case scenario.

In each of the sections below, we should already list known cases that need to be tested in the final implementation, and at which level. The initial version here should be a best of effort: it is perfectly acceptable and expected that this section will be amended during implementation.

### Unit
### Integration
### E2E
-->

# Drawbacks
[drawbacks]: #drawbacks

* Increases complexity of solution
* Offloading large data to external system will (likely) prevent us from using the data in FEEL expressions

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

The proposal is inspired by Git LFS (https://git-lfs.github.com/). The author has not had personal experiences in using Git LFS.

# Out of scope
[out-of-scope]: #out-of-scope

Call out anything which is explicitly not part of this ZEP.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Does the idea have merit?
- Which implementation approach to pursue? 
- How to handle eviction of data in external volume?

<!--
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this ZEP that could be addressed in the future independently of the solution that comes out of this ZEP?
-->
# Future possibilities
[future-possibilities]: #future-possibilities
<!--
Think about what the natural extension and evolution of your proposal would be and how it would affect the language and project as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the ZEP you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future ZEP; such notes should be in the section on motivation or rationale in this or subsequent ZEPs. The section merely provides additional information.


-->