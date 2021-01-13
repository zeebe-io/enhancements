---
title: ZEP Template
authors:
  - TBD
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: yyyy-mm-dd
last-updated: yyyy-mm-dd
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
see-also:
  - "/zeps/ZEP-0001.md"
replaces:
  - "/zeps/ZEP-0002.md"
superseded-by:
  - "/zeps/ZEP-0003.md"
---

# Summary
[summary]: #summary

<!--
One paragraph summary of the feature
-->

# Motivation
[motivation]: #motivation

<!--
- [ ] Why are we doing this?
- [ ] What problem are we solving?
- [ ] What is the expected outcome? 
-->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

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
-->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!--
This is the technical portion of the ZEP. After reading it, a contributor should understand/know the following:

- [ ] The impact of the changes on other features is clear.
- [ ] The implementation is delineated
- [ ] Known corner cases are listed and addressed

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.


- [ ] Does the ZEP affect the official Zeebe distribution (e.g. configuration, logging)?
- [ ] Does the ZEP require coordination with the platform team?
- [ ] Does the ZEP require coordination with the Operate team?
-->

## Compatibility

<!--
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
-->

## Testing

<!--
You should describe what is the overall functionality that should be tested.

If you are omitting tests, explain why, and explain the impact if it fails, specifically the worst case scenario.

In each of the sections below, we should already list known cases that need to be tested in the final implementation, and at which level. The initial version here should be a best of effort: it is perfectly acceptable and expected that this section will be amended during implementation.

### Unit
### Integration
### E2E
-->

# Drawbacks
[drawbacks]: #drawbacks

<!--
Why should we *not* do this?
-->

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!--
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
-->

# Prior art
[prior-art]: #prior-art

<!--
Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

- For language, library, tools, and UI proposals: Does this feature exist in other tools/products and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your ZEP with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other products is some motivation, it does not on its own motivate a ZEP.
-->

# Out of scope
[out-of-scope]: #out-of-scope

<!--
Call out anything which is explicitly not part of this ZEP.
-->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!--
- What parts of the design do you expect to resolve through the ZEP process before this gets merged?
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
