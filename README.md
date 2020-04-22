# Enhancements

Zeebe Enhancement Proposals (ZEPs) are a method we use in the design phase to validate ideas and explore the solution space. It's similar to an RFC, but more general purpose.

## When

There is no hard rule on when something requires a ZEP, but here are some indicators that it might:

- there are several possible solutions and it's not immediately clear which one is best
- it might be hard to understand your design and why you chose it if you weren't there to explain it
- the implementation will require changes by multiple people or teams
- explaining the entire solution would take more than a README
- it affects or implements "business logic"
- it would be hard to rip out and replace if you weren't around
- there is discussion or disagreement about the ideal solution

## Process

### New ZEPs

1. clone this repo, or fork it if you don't have the necessary permissions
1. copy the `TEMPLATE.md` to `ZEP000-[short-name].md`
1. open a PR when you're ready. You can prefix the title with `[WIP]` to let people know it's a work in progress but solicit feedback, or collaborate with coauthors.
1. once a PR is reviewed and approved:
  - assign a ZEP number
    - rename the file
  - update the yaml at the top with the right reviewers, approvers, etc.
  - merge the PR

### Changes

1. For `provisional` or `implementable` ZEPs, open a PR with the changes you'd like to make. Where possible, get review from the original authors and approvers.
1. For `implemented` ZEPs (i.e. already shipped), open a new ZEP and add the old one to the `see-also` or `replaces` section. Feel free to refer to specific sections you are superseding, or PR to add back references to the old ZEP if that seems important.
1. For major changes, consider superseding the prior one by making a new ZEP and changing the status of the old one to `replaced`

### Linking

- link tickets which directly relate to implementation of ZEPs, ideally to the commit/PR of the ZEP the ticket is based on (in case it changes later).
- it's ok to use the ZEP as canonical documentation for features too, but follow the process above for changes. This is a good reason to keep them high-level.

### Lifecycle details

Refer to the rust [docs](https://github.com/rust-lang/rfcs#the-rfc-life-cycle) for details of what the process options are. Register any major differences in the future as PRs to this section of this file.

## Tools

https://hackmd.io/ is an option. In the early draft stages you can also use gdocs to collaborate quickly.

## Prior art

Inspired by [IETF I-D/RFCs](https://ietf.org/standards/ids/), [Python PEPs](https://www.python.org/dev/peps/), and [Kubernetes KEPs](https://github.com/kubernetes/enhancements/blob/master/keps/0001-kubernetes-enhancement-proposal-process.md), we borrowed heavily from the [Rust RFC](https://github.com/rust-lang/rfcs) template for this one.
