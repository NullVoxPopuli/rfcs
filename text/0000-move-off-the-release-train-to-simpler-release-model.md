---
stage: accepted
start-date: 2026-06-05T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
  - steering
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite:
---

# Move off the release train to a simpler release model

## Summary

Ember's "release train" today bundles two separate things:

1. **A six-week release cadence.** A new minor ships on a steady, predictable
   clock.
2. **A multi-channel promotion pipeline.** Code flows through dedicated
   `canary` → `beta` → `release` branches, cut and promoted by a rotating
   release manager working a mostly-manual checklist each cycle.

**This RFC keeps (1) and replaces (2).** The six-week cadence is the part that
works, and it stays exactly as it is. What goes away is the multi-channel branch
machinery and the hand-run release ceremony.

Concretely:

- A single long-lived branch: **`main`**. No `canary`, `beta`, or `release`
  branches.
- Stable releases are cut from `main` by [`release-plan`][release-plan] on the
  same six-week schedule, with the `npm publish` gated behind a protected
  [GitHub deployment environment][gh-environments] — a required reviewer
  approves before it publishes.
- The separate, published **`canary` and `beta` channels go away**. "Bleeding
  edge" becomes *track `main` via git*: consumers who want unreleased code point
  at the git ref rather than a special npm dist-tag maintained off a special
  branch. `main` itself is the canary/alpha.

SemVer, the six-week cadence, the deprecation policy, LTS, and the major-version
process from RFC [#0830][rfc-830] are all **unchanged**. Only the branch
structure and the publishing mechanics change.

[release-plan]: https://github.com/embroider-build/release-plan
[gh-environments]: https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment
[rfc-830]: https://github.com/emberjs/rfcs/blob/master/text/0830-evolving-embers-major-version-process.md

## Motivation

The expensive part of the release train is not the cadence — it's the machinery
around it.

### Three live channels are a standing branch-management cost

Maintaining `canary`, `beta`, and `release` as separate, simultaneously-live
lines (plus LTS) means ongoing cherry-picks, back-merges, and branch bookkeeping
that exist only to service the shape of the train. Every cycle a release manager
(RM) works a long, mostly-manual checklist: cut `beta` from `canary`, promote
`beta` to `release`, hand-publish `ember-source`, `ember-cli`, and (historically)
`ember-data` in lockstep, regenerate and proofread the changelog, and coordinate
the release. That work:

- **Needs a volunteer on a clock, indefinitely**, and slips when no RM is
  available. It is a recurring source of burnout and bus-factor risk.
- **Is manual and therefore error-prone** — version bumps, changelog assembly,
  and multi-package lockstep publishing are exactly the mechanical steps tooling
  does more reliably than a person at the end of a checklist.

### The ecosystem already automated this

Nearly every modern Ember addon releases with [`release-plan`][release-plan]. It
derives the SemVer bump and changelog from merged PRs and publishes to npm with
provenance, no scheduled human ceremony required. The framework does not need a
bespoke, hand-run process that is strictly more work than the tooling everyone
else already trusts. **We can get away with `release-plan`.**

### `canary`/`beta` as published channels mostly duplicate "the git repo"

The reason `canary` exists is so people can run unreleased Ember. But that is
exactly what the default branch *is*. Publishing it as a separate, branch-backed
npm channel is extra plumbing to approximate "whatever is on `main` right now."
Letting consumers track `main` via git directly removes the plumbing without
removing the capability.

### Keep the cadence — it works

The six-week cadence is predictable, well-understood, and well-loved. It is *not*
what makes the train expensive, so this proposal deliberately keeps it. We are
moving off the *channel-and-ceremony* model, not the calendar.

## Detailed design

### Branching model

A single long-lived branch: **`main`**. All pull requests merge here. `main` is
both where work integrates and the source of every release. There are no
`canary`, `beta`, or `release` branches, and none of the cherry-pick /
back-merge machinery that keeping three live channels requires.

### Per-PR release metadata

Every PR that changes shipped code declares:

1. **A SemVer impact** — `patch`, `minor`, or `breaking` (major) — via a label
   (or a changeset file, whichever the team prefers; `release-plan` supports the
   label-driven flow out of the box).
2. **A changelog entry** — the human-readable "what changed," authored as part
   of review rather than reconstructed afterward.

`release-plan` aggregates these across everything merged since the last release
to compute the next version number and assemble the changelog. The decision a
release manager used to make by reading the diff is now made, deterministically,
from metadata that was reviewed when the change landed.

Breaking changes (`breaking` label) still require the same approval they require
today: a `major` is a steering-level decision, sequenced by the major-version
process (RFC #0830), not something an automated bump performs silently.

### Releasing: the six-week schedule, automated and gated

The cadence is unchanged; the *mechanics* are automated:

1. `release-plan prepare` keeps a release PR continuously up to date — it
   computes the pending version and changelog from the metadata of everything
   merged since the last release.
2. On the scheduled six-week release date, that release PR is merged, which
   triggers the publish workflow. (The schedule is the rhythm; nothing forces a
   release between scheduled dates, and a date can still be held or moved by the
   same people who manage the calendar today.)
3. The publish job targets a **protected GitHub Environment** (e.g.
   `npm-publish`) with a *required reviewers* protection rule. The npm token /
   trusted-publishing identity is scoped to that environment, so nothing can
   publish until a required reviewer clicks **Approve** on the pending
   deployment.
4. On approval, the job runs `release-plan publish`: it tags, pushes, and
   publishes to npm (with provenance via OIDC trusted publishing).

The deliberate approval click is the entire residual ceremony — there is no
checklist, no manual version edit, no manual changelog, no manual `npm publish`,
and no channel to cut or promote. Because the gate is a GitHub Environment, the
existing GitHub permission and audit model applies: who may approve, the record
of who approved what, and required-reviewer rotation are all standard repo
configuration rather than tribal release-manager knowledge.

### Channels

- **Stable (`latest`)** — published from `main` through the gated environment,
  on the six-week cadence, as above.
- **Canary / alpha** — **is `main`.** To run unreleased Ember, depend on the git
  ref (e.g. `main` or a specific commit) rather than a published `canary`
  dist-tag. There is no longer a separately-managed canary branch or its nightly
  publish job; the latest in-progress code is just the head of `main`.
- **`beta`** — removed. The dedicated beta channel and its branch go away. A
  specific change that warrants extended baking can still be merged early and
  exercised via the git-tracked `main` before the next scheduled stable release;
  it just isn't a permanent, separately-published channel.

> How `ember-source` consumers consume `main` from git — since the published
> package is normally a build artifact — is the main detail to settle. See
> [Unresolved questions](#unresolved-questions).

### Deprecations, majors, and LTS — unchanged

Because the six-week cadence is retained, the policies layered on top of it are
**unaffected**:

- **Deprecations** are still introduced as SemVer-minor and removed in majors,
  on the same deprecation-freeze schedule.
- **Majors** still follow the major-version process of RFC #0830 (its `M.10`
  deprecation freeze and `M.12` → `(M+1).0` train). That process is sequenced by
  the minor cadence, which this proposal keeps; it simply *executes* via
  `release-plan` from `main` instead of via the channel pipeline.
- **LTS** continues as today.

In other words, this RFC changes *how* a release is cut and *which branches/
channels exist*, not *when* releases happen or *what compatibility they
guarantee*.

### Lockstep across packages

Historically `ember-source`, `ember-cli`, and `ember-data` released in lockstep.
`release-plan` operates per repository. This proposal does **not** mandate
dropping lockstep, but it makes lockstep an explicit, opt-in coordination step
rather than a side effect of the channel pipeline:

- If lockstep is kept, the release workflow coordinates the version across the
  packages at release time.
- If lockstep is relaxed, normal SemVer ranges already express cross-package
  compatibility, and each package releases on its own schedule.

This is called out as a design consideration rather than decided here; see
[Unresolved questions](#unresolved-questions).

### What is removed

- The published `canary` and `beta` channels and their dedicated branches.
- The cherry-pick / back-merge machinery for keeping three live channels.
- The release-manager checklist: manual version bumps, manual changelog
  assembly, manual lockstep `npm publish`, and channel cut/promote steps.

### What is kept

- **The six-week release cadence**, unchanged.
- SemVer and every existing compatibility guarantee.
- The deprecation policy, LTS, and the major-version process (RFC #0830).
- Steering control over majors and over what ships.
- A deliberate human gate before each publish.

## How we teach this

**Contributors** learn one habit: every PR that changes shipped code gets a
SemVer-impact label and a changelog entry, authored during review. This is the
same workflow they already follow in virtually every addon.

**Release approvers** (a documented, rotating group with access to the protected
environment) review the pre-computed release PR and approve the deployment. The
role shrinks from "run the channel/checklist ceremony" to "review and click
approve."

**Consumers of `canary`/`beta`** are the most important audience for this
change, because their workflow moves. Anyone who today depends on
`ember-source@canary` or `ember-source@beta` — including `ember-try` scenarios,
addon CI matrices that test against upcoming Ember, and people bisecting
regressions — needs to switch to tracking `main` via git. The migration guide
must spell out exactly how to do that (see Unresolved questions), and the
default `ember-try` / blueprint scenarios should be updated to the new approach.

Documentation work:

- Rewrite the website Releases page: `main` is the release line, the six-week
  cadence is unchanged, and there are no longer published `canary`/`beta`
  channels — track git for the edge.
- Update the contributor guide with the label/changelog workflow.
- Document the environment, the approver group, and the approval procedure.
- A migration/announcement blog post focused on the `canary`/`beta` → git change.

## Drawbacks

- **Losing published `canary`/`beta` builds is a real regression for some
  workflows.** `npm install ember-source@beta` is lower-friction than depending
  on a git ref, and a lot of ecosystem CI (`ember-try`, addon test matrices) is
  built around those dist-tags. Tracking `main` from git must be made genuinely
  easy or this trades maintainer effort for downstream effort.
- **`ember-source` is a build artifact**, so "just depend on the git ref" is not
  as turnkey as it is for a typical source-published addon. The consumption story
  for `main` needs to be concrete and supported, not hand-waved.
- **Per-PR labeling discipline.** A wrong impact label yields a wrong bump.
  `release-plan` makes the bump deterministic, but the input is human-supplied.
- **Concentrated publish authority.** Publish power moves to whoever can approve
  the protected environment; that group's security and rotation matter.
- **Tooling dependency.** The framework's release process becomes coupled to
  `release-plan` — a small, community-owned tool, but a new dependency.
- **Cultural change.** `canary` and `beta` are long-standing, load-bearing parts
  of Ember's testing culture and infrastructure; removing them as channels is not
  only a mechanical change.

## Alternatives

- **Keep the train as-is (status quo).** Pays the recurring branch-management and
  RM-ceremony cost, and the bus-factor risk, indefinitely.
- **Keep a published `canary` (or `beta`), but automate it.** A middle ground:
  drop only `beta`, and have `release-plan` publish a `canary` prerelease
  dist-tag off `main` automatically so existing `@canary` consumers don't have to
  move to git. This preserves the convenient dist-tag at the cost of keeping one
  prerelease channel. Worth considering if the git-tracking story proves too
  rough for downstream CI.
- **Use `changesets` instead of `release-plan`.** Functionally similar;
  `release-plan` is preferred because the Ember ecosystem has standardized on it.
- **Also drop the cadence (fully continuous releases).** A more radical model
  where every merge can release. Explicitly **not** proposed here — the six-week
  cadence is retained deliberately because it works.

## Unresolved questions

- **How consumers track `main` via git.** Given `ember-source` is a built
  artifact: is it a git dependency on a build branch, an automated prerelease tag
  cut from `main`, published nightly git snapshots, or documented local-build
  instructions? This is the crux of the proposal and must be settled.
- **Whether any prerelease dist-tag is retained at all** (the "automate canary"
  alternative) versus purely git-tracked `main`.
- **`ember-try` and ecosystem CI migration** off `@canary`/`@beta` scenarios.
- **Lockstep.** Do `ember-source`, `ember-cli`, `ember-data`/WarpDrive keep
  lockstep versioning, or release independently? If lockstep, where does the
  coordination live in the workflow?
- **Approver group.** Who can approve the protected environment, how membership
  rotates, and the security posture (2FA, OIDC trusted publishing, audit).
