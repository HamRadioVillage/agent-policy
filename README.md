# agent-policy

Central policy for the HamRadioVillage fix agent. A labelled issue in any
participating repo runs the action in this repo, which triages the report,
fixes it if it can find a root cause, documents the change, and opens a pull
request for a human to review.

Three files matter:

- `.github/actions/agent-fix/action.yml` — the composite action callers use
- `agent/triage-and-fix.md` — the playbook the agent follows
- `callers/TEMPLATE.yml` — the workflow a participating repo copies

`callers/TEMPLATE.yml` is a template, not one of this repo's own workflows,
which is why it sits in `callers/` and not `.github/workflows/` — GitHub
would otherwise try to register it as a workflow here.

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
`permissions`, `concurrency` and `timeout-minutes`; `callers/TEMPLATE.yml`
carries them so that stays copy-paste.

It also removes a hack: the action's files are already on disk at
`$GITHUB_ACTION_PATH`, so the playbook needs no second checkout and cannot
end up inside the agent's `git add -A`.

## Calling it

Copy [`callers/TEMPLATE.yml`](callers/TEMPLATE.yml) to
`.github/workflows/agent-fix.yml` in your repo and edit its three marked
sections: your toolchain, the pinned SHA, and `allowed_tools`. That file is
the only one a participating repo needs, and it is kept here rather than
summarised here so there is one copy to keep correct.

**Pin to a commit SHA, never `@main` or a tag.** This action runs with write
access to the calling repo and can mint Anthropic tokens on the org's behalf,
which makes this repo the highest-value target in the org. A mutable ref
means whoever can move it decides what runs in your repo. Treat the ref like
a dependency version: `main` is protected, CODEOWNERS requires review, and
callers pin.

The template ships an unresolvable `@REPLACE_WITH_SHA` rather than a real
commit, on purpose. A template that lives inside the repo it pins can never
hold a current SHA — the commit that updates the pin becomes the new HEAD —
so any real value here would be stale the next time the action changes, and
a stale pin resolves and runs perfectly happily. Failing on an unresolvable
ref costs one deliberate step and cannot pass silently.

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

A denial that never reaches that tab was never a denial: the exchange did not
happen. The action's first step therefore fails the run when any `anthropic_*`
input is empty, naming both the input and the variable to set. That check
earns its place because **GitHub does not enforce `required` on composite
action inputs** — it is documentation, not validation — and because a repo
reads an org variable as empty, rather than failing, in two different cases:
the variable does not exist, or its visibility does not include that repo.
The second is invisible from the repo side. Nothing is misspelled, nothing
errors, and the run fails with the same opaque `401` as a real claim
mismatch, which sends you reading auth history for an entry that was never
written. The step also warns, without failing, when an identifier does not
carry its expected prefix — that catches two variables pasted into each
other's slots, while leaving Anthropic free to change the formats.

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

## Forks

If the calling repo is a fork, `gh` resolves its default repository to the
**parent**, not the fork. Unpinned, `gh issue view 7` reads the parent's
issue 7 and `gh pr create` opens the PR on the parent's repo. The action sets
`GH_REPO` to the calling repo so that cannot happen by accident, and the
prompt passes the PR target explicitly.

Two other fork defaults bite before any of this runs: a fork has **Issues
disabled** and **Actions disabled** until someone turns each on. Without
Issues there is nothing to label, so the trigger can never fire.

To deliberately send a fork's work upstream, set `pr_target_repo`:

```yaml
    pr_target_repo: upstream-owner/name
```

The PR is then opened cross-fork — from your branch against their base —
while every issue comment and label stays on your fork. Understand what that
means before switching it on: each run becomes a public contribution to a
project you do not own, under your org's name, originating from a bug report
written by whoever was in Discord. Section 5 of the playbook makes the agent
read the target's `CONTRIBUTING.md`, stop if they have a policy against
AI-generated contributions, disclose what it is, and keep the reporter's
prose and identity out of the PR entirely.

One consequence worth planning for: a PR on someone else's repo raises no
webhook this org receives, so the only thing that reaches the Discord thread
is the agent's comment on the originating issue.

## Changing the playbook

`agent/triage-and-fix.md` is the product. When a PR comes back wrong, the fix
usually belongs there rather than in a caller — and it ships to every
participating repo at once, as soon as they move their pinned SHA.

A repo can add `.github/agent/overrides.md` for its own conventions; it is
read in addition to the playbook and wins on conflicts.
