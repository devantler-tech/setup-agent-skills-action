# Setup Agent Skills

Install agent skills with the [`gh skill`](https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli/) CLI from a newline-separated list of `<owner/repo> <skill>[@pin]` entries, for **one or more agents** in a single step (e.g. GitHub Copilot and Claude Code).

`gh skill install` writes upstream provenance (`metadata.github-*`) into each installed `SKILL.md`, so no lockfile is required — checked-in skills are self-describing and [`update-agent-skills`](https://github.com/devantler-tech/update-agent-skills-action) or `gh skill update --all` picks up drift natively.

## Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `skills` | Newline list of `<owner/repo> <skill>[@pin]`. `@pin` is an optional tag/branch/SHA. `#` comments and blank lines are allowed. | ✅ | — |
| `agents` | One or more agents passed to `gh skill install --agent`, separated by whitespace, commas, or newlines (e.g. `github-copilot claude-code`). Each skill is installed once per agent. See `gh skill install --help` for the full list of supported agents. | ❌ | `github-copilot` |
| `scope` | Value passed to `gh skill install --scope` (`project` or `user`) | ❌ | `project` |
| `gh-version` | Minimum required `gh` version (must support `gh skill`) | ❌ | `2.90.0` |
| `github-token` | GitHub token exposed to `gh` as `GH_TOKEN` | ❌ | `${{ github.token }}` |
| `experimental-rate-limit-retry` | Opt in to five install attempts with waits of 60, 120, 240, and 240 seconds (eleven minutes total, plus command runtime). Off keeps the shared fast retry defaults. See [Rate-limit retries](#rate-limit-retries). | ❌ | `false` |

## Rate-limit retries

GitHub recommends waiting at least one minute after a secondary rate limit when
response headers are unavailable, followed by exponential backoff.
The opt-in follows that [fallback policy](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api#exceeding-the-rate-limit),
with a four-minute cap per wait and five total attempts. A successful install
returns immediately; an exhausted retry sequence fails with the CLI's exit status.
The policy applies to all failed skill-install commands, since the CLI does not
expose the response headers needed to distinguish rate-limit classes reliably.

This option does not guarantee recovery from a primary hourly quota or honor a
server-provided reset deadline that the CLI does not expose. Keep it disabled
until real burst recovery is evaluated for the consumer, as tracked in
[#288](https://github.com/devantler-tech/.github/issues/288).

## Outputs

| Name | Description |
|------|-------------|
| `installed-skills` | Newline-separated list of `source/skill-name[@pin]` entries that were installed (listed once per skill, regardless of how many agents it was installed for) |

## Usage

```yaml
steps:
  - uses: actions/checkout@<full-commit-sha> # vX.Y.Z
  - uses: devantler-tech/setup-agent-skills-action@<full-commit-sha> # vX.Y.Z
    with:
      # Install every skill for both Copilot and Claude Code in one step.
      agents: |
        github-copilot
        claude-code
      skills: |
        github/awesome-copilot git-commit
        fluxcd/agent-skills gitops-knowledge
        devantler-tech/agent-plugins plugins/engineering-practices/skills/ways-of-working@dc62d7fc09d62602d45da43e31cb1c3998bf05c5
        # Pin with @<tag|branch|sha> or omit to track the upstream default branch.
```

Omit `agents` to install for GitHub Copilot only (the default), or set it to any single agent (e.g. `claude-code`).

## Agent selection

The `agents` input accepts one or more supported agents. The same installed
`SKILL.md` content works across the agents supported by `gh skill`.

## Requirements

- A trusted runner-provided `gh` with `gh attestation` support. If it is older than **2.90.0** (and therefore lacks `gh skill`), this action downloads the required GitHub CLI release archive on Linux or macOS (amd64/arm64) and verifies its GitHub artifact attestation before installation. Windows runners must pre-install `gh >= 2.90.0`.

## Release recovery

Releases run from protected `main`. To preview the next release without writing
anything, dispatch the `Release` workflow on `main` with `dry-run` enabled:

```sh
gh workflow run release.yaml --ref main -f dry-run=true
```

A dry run previews semantic-release's next version or reports that no release is
needed. It does not recreate a missing GitHub release for an existing tag.
If a release fails before creating its tag, repair the failure and rerun the
workflow from `main`.

If the tag was pushed but its GitHub release is missing, first verify that the
existing tag resolves to the intended reviewed commit on protected `main` and
that its CI passed. In the repository's **Releases** page, choose **Draft a new
release**, select that **existing tag**, generate and review its release notes,
and publish. Select **Publish this Action to the GitHub Marketplace** if the
release also needs to be listed there. Confirm the published release's source
commit and Marketplace version afterward. Do not move or delete the tag, or
try to edit an immutable release; use a new patch release at the same reviewed
commit if a published immutable release needs a separate Marketplace listing.
