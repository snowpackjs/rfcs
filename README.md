# Astro Project Roadmap

## Overview

- [Glossary](#glossary)
- [Stage 1: Proposal](#stage-1-proposal)
- [Stage 2: Accepted Proposal](#stage-2-accepted-proposal)
- [Stage 3: RFC \& Development](#stage-3-rfc--development)
- [Stage 4: Ship it!](#stage-4-ship-it)

## Glossary

> **Proposal Champion:** A proposal is more likely to move forward in this process if it has a **champion** attached to it. This role is self-nominated and open to anyone (both maintainers and community members are welcome to volunteer). It may be the original proposal author, or someone who joins later. The responsibility of a champion is to help shepard the proposal through the later parts of this process, including: writing the detailed RFC, responding to and incorporating feedback, and eventually implementing the proposal in code.
>
> **You are not alone as a champion!** If this is your first time writing an RFC or design document, our maintainer team is expected to work with you and guide you through this process.

## Stage 1: Proposal

**Goal:** Unstructured, low-friction conversations on ideas and improvements to Astro. Useful for gathering early feedback and gauging interest with the community and maintainers.

**Requirements:** None! To suggest an improvement, [create a new Discussion](https://github.com/withastro/roadmap/discussions) using our (completely optional) [proposal template](stage-1--discussion-template.md?plain=1).

**Location:** GitHub Discussions [(see all open proposals).](https://github.com/withastro/roadmap/discussions) The Astro Discord channel `#feedback-ideas` can also be used to throw an idea out for quick initial feedback, but be warned that chat is short-lived and not designed for longer-lived discussion.

## Stage 2: Accepted Proposal

**Goal:** Confirm proposal feasibility with Astro Maintainers and the [Technical Steering Committee (TSC)](https://github.com/withastro/.github/blob/main/GOVERNANCE.md#technical-steering-committee-tsc).

**Requirements:** An existing proposal (Stage 1). In addition, a proposal is more likely to be accepted if it is detailed and well thought-out, can demonstrate community interest, has at least one champion volunteer, has a Maintainer (L2+) acting as a champion, and has buy-in/interest from Astro maintainer(s).

**Location:** GitHub Issues [(see all accepted proposals).](https://github.com/withastro/roadmap/issues)

**What to Expect:**

RFCs advance through the stages during RFC meetings with TSC and maintainers. Voting follows the RFC proposal voting process.

Meetings occur ad hoc rather than on a scheduled basis. They are called for when a proposal author or champion feels it is ready to advance to the next stage. The author or champion can ask for a meeting by contacting a [Core member](https://github.com/withastro/.github/blob/main/MEMBERS.md#core) to schedule a time.

All maintainers are invited to the meeting. If consensus is reached, the RFC advances to the next stage. See RFC proposal documentation for full details.

When a proposal is accepted, a maintainer (L2+) will create a new GitHub Issue summarizing the original proposal using our official template.

The proposal's original author will be given a chance to volunteer to be its champion. If they refuse, someone else can volunteer for the position by posting in the GitHub Issue. An accepted proposal will remain an open issue until a champion is confirmed by TSC. Once confirmed, the champion is free to advance the proposal to the next stage.

In some cases, a proposal may be explicitly rejected by TSC if it is known to be infeasible, or go against some existing goals/mission of the project. In the event of an explicit rejection, a TSC member will comment on to the proposal explaining the reasoning for rejection.

A stale, accepted proposal can be removed (rejected after a previous acceptance) at any time following the same, existing [RFC Proposal](https://github.com/withastro/.github/blob/main/GOVERNANCE.md#voting-rfc-proposals) voting process.

## Stage 3: RFC & Development

**Goal:** Begin development! Gather implementation feedback and work with maintainers during development.

**Requirements:** An accepted proposal (Stage 2) and a proposal champion to author and implement the RFC.

**Location:** GitHub Pull Requests [(see all in-progress RFCs)](https://github.com/withastro/roadmap/pulls) [(see all finished RFCs)](https://github.com/withastro/roadmap/tree/main/proposals)

**What to Expect:** To create an RFC for an already-accepted proposal, the proposal champion must use our [`stage-3--rfc-template.md`](stage-3--rfc-template.md?plain=1) RFC template in the repo. The initial sections of the RFC template should be copy-pasted from the the accepted proposal (they match 1:1). All remaining sections are left for the champion to complete with the implementation and tradeoff details of the RFC.

When a Stage 3 RFC is opened, the Stage 2 RFC should be closed with a comment that links to the Stage 3 RFC pull request.

You do not need to get an RFC approved before beginning development! One of the best ways to validate your RFC is to prototype, so early prototyping and parallel development alongside the RFC is strongly encouraged. The RFC is a living document during this stage, and is most useful for gathering feedback as you build. An RFC will not be accepted and merged until it's PR is also ready to merge.

The proposal champion can request feedback on their RFC at any point, either asynchronously in Discord (inside the `#dev`/`#dev-ptal` channel) or during our weekly community call. Maintainers are expected to provide timely feedback at this stage so that the RFC author is never blocked. If you are an RFC champion and need access to the `#dev-ptal` channel, message **@fks** for permission.

## Stage 4: Ship it!

An RFC is ready to be approved and finalized once it's Pull Request is ready for its final review. When a champion thinks the RFC is ready he can ask for a call for consensus.

At this time, some member of the core team will motion for a final comment period (FCP). This follows our existing [RFC Proposal](https://github.com/withastro/.github/blob/main/GOVERNANCE.md#voting-rfc-proposals) voting process. Once the final comment period has elapsed the RFC will be merged if there are no objections.

## RFC Meetings

RFCs advance through the stages during RFC meetings with TSC and maintainers. Voting follows the [RFC proposal](https://github.com/withastro/.github/blob/main/GOVERNANCE.md#voting-rfc-proposals) voting process.

Meetings occur ad hoc rather than on a scheduled basis. They are called for when a proposal author or champion feels it is ready to advance to the next stage. The author or champion can ask for a meeting by contacting TSC to schedule a time.

All maintainers are invited to the meeting. If consensus is reached, the RFC advances to the next stage. See [RFC proposal](https://github.com/withastro/.github/blob/main/GOVERNANCE.md#voting-rfc-proposals) documentation for full details.

---

**Prior Art / Special Thanks**

This process is an amalgamation of [Remix's Open Development process](https://remix.run/blog/open-development) and our previous [RFC process](https://github.com/withastro/roadmap/blob/78b736c28fe487ad02ec76bb038ad1087c471057/README.md), which had been based on the RFC processeses of the Vue, React, Rust, and Ember projects.
