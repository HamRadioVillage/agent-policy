# agent-policy

Central policy for the HamRadioVillage fix agent. A labelled issue in any
participating repo runs the action in this repo, which triages the report,
fixes it if it can find a root cause, documents the change, and opens a pull
request for a human to review.

Two files matter:

- `.github/actions/agent-fix/action.yml` — the composite action callers use
- `agent/triage-and-fix.md` — the playbook the agent follows

## Why this repo is public

A public repository cannot call a reusable workflow or an action stored in a
private repository, and the org-level Actions **Access** setting does not
change that — it only governs private callers. A private policy repo would
silently lock out `urfd`, `villagers` and every other public repo in the org.

Nothing secret lives here, and nothing secret is passed in. The action
authenticates to the Claude API with Workload Identity Federation: the job
exchanges its GitHub OIDC token for an Anthropic token that expires in
minutes. There is no API key in this repo, in any caller, or on any runner.

## Why a composite action and not a reusable workflow

A reusable workflow owns the whole job, so the caller cannot run setup steps
of its own — it can only pass a shell snippet. That breaks on any repo whose
toolchain needs an action: `villagers` pins `ruby-3.4.1`, which is not on the
runner image, so it needs `ruby/setup-ruby@v1` before the agent starts.

A composite action lets each caller set its toolchain up idiomatically and
then hand off. The tradeoff is that the caller declares its own
`permissions`, `concurrency` and `timeout-minutes`; the template in the
pipeline repo carries them so that stays copy-paste.

It also removes a hack: the action's files are already on disk at
`$GITHUB_ACTION_PATH`, so the playbook needs no second checkout and cannot
end up inside the agent's `git add -A`.

## Calling it

```yaml
permissions:
  contents: write
  issues: write
  pull-requests: write
  id-token: write          # required: the agent's auth is keyless

# ...

- uses: actions/checkout@v4
  with: { fetch-depth: 0 }

# ... your toolchain ...

- uses: HamRadioVillage/agent-policy/.github/actions/agent-fix@<sha>
  with:
    issue_number: ${{ github.event.issue.number }}
    anthropic_federation_rule_id: ${{ vars.ANTHROPIC_FEDERATION_RULE_ID }}
    anthropic_organization_id: ${{ vars.ANTHROPIC_ORGANIZATION_ID }}
    anthropic_service_account_id: ${{ vars.ANTHROPIC_SERVICE_ACCOUNT_ID }}
    allowed_tools: >-
      Read,Write,Edit,Glob,Grep,
      Bash(git:*),Bash(gh issue:*),Bash(gh pr:*)
```

**Pin to a commit SHA, never `@main`.** This action runs with write access to
the calling repo and can mint Anthropic tokens on the org's behalf, which
makes this repo the highest-value target in the org. Treat the ref like a
dependency version: `main` is protected, CODEOWNERS requires review, and
callers pin.

## Authentication

Keyless, via Workload Identity Federation. The three `anthropic_*` inputs are
identifiers rather than credentials, so they belong in `vars`, not `secrets` —
they are safe to read in a public repo. The job presents its GitHub OIDC
token, Anthropic matches it against a federation rule you configure in the
Console, and mints a token that expires in minutes.

There is deliberately no `anthropic_api_key` input. A static key outranks
federation in the SDK's credential chain, so accepting both would let a stray
key silently shadow the federated path — you would think federation worked
when it did not.

**Each participating repo gets its own federation rule, pinned to that repo
and its default branch:**

```json
{
  "match": {
    "subject_prefix": "repo:HamRadioVillage/villagers:ref:refs/heads/main",
    "audience": "https://api.anthropic.com",
    "claims": { "repository_owner": "HamRadioVillage" }
  }
}
```

The rule *is* the opt-in list, and deleting it is the kill switch for that
repo. Do not widen `subject_prefix` to `repo:HamRadioVillage/*`: that matches
every repo in the org, including ones added later, and without a `ref`
constraint it also matches pull requests from forks — anyone who can open a PR
would get a token. The `issues: labeled` trigger always runs on the default
branch, so the exact prefix above is sufficient and is the narrowest form.

When an exchange is denied the API returns an opaque `401`; the reason is on
the authentication-history tab of the Console's workload-identity page.
`match_subject_prefix` means the rule is narrower than the run that fired.

`allowed_tools` is the per-repo safety boundary. Bash is off by default in
`claude-code-action`, so any command not listed there cannot be executed —
list the repo's real test and lint commands and nothing more.

## What the agent will and will not do

It triages before touching anything, and ends with a comment and a
`needs-info` label instead of a PR when the report is too vague, is a feature
request, has no identifiable root cause, would sprawl across modules, or looks
security-sensitive. Finishing without a PR is a successful outcome.

It never merges, never enables auto-merge and never closes the issue. When it
cannot get the repo's own checks green it opens a draft PR and says so.

It treats the report body as untrusted data. Reports arrive from Discord via
radar, quoted verbatim; rule zero of the playbook tells the agent to ignore
and surface any instructions embedded in them.

## Changing the playbook

`agent/triage-and-fix.md` is the product. When a PR comes back wrong, the fix
usually belongs there rather than in a caller — and it ships to every
participating repo at once, as soon as they move their pinned SHA.

A repo can add `.github/agent/overrides.md` for its own conventions; it is
read in addition to the playbook and wins on conflicts.
