# Release runbook

The steps every release follows. A release issue in the milestone ticks them off and records what
that release found. Change this file only when the process changes.

## Before

- [ ] Every issue in the milestone is closed or moved out of it.
- [ ] The milestone description still states the release's goal, acceptance, and what is out.
- [ ] `CHANGELOG.md`'s Unreleased section reads as the release notes.

## Staging release

- [ ] Cut a staging release from a clean `main`.
- [ ] Its artifacts are at their URLs, and the checksums match.

## Acceptance

<!-- The clean-machine checks this project requires, per platform. -->

- [ ] ...

A failure files an issue in the milestone and stops the release until it is fixed.

## Cut and publish

- [ ] Tag from a clean `main`.
- [ ] Publish, and check the published artifacts anonymously.
- [ ] Announce.

## After

- [ ] Close the release issue and the milestone.
