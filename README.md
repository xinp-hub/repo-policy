<!-- exported from FORGE templates/policy; edit it there, not here -->
# repo-policy

Composite GitHub Actions that decide which pull requests may merge without a human, ask an
independent reviewer about the small ones, and keep AI coding agents out of a repository's
contributor list. Python 3 standard library only; no third-party actions.

| Action | Runs | Output |
|---|---|---|
| `classify` | `scripts/merge_policy.py` | `tier`: 0, 1 or 2 |
| `review` | `scripts/agent_review.py` | `approved`: `true` or `false` |
| `attribution` | `scripts/ai_attribution.py` | fails when a commit credits an AI agent |

This repository is a copy. The code and its tests live in FORGE (`templates/policy/`,
`tests/test_merge_policy.py`, `tests/test_policy_actions.py`), and
`scripts/export_policy.py` writes this tree. A change made here is overwritten by the next
export; make it in FORGE.

## Using it

Pin every action by full commit SHA, with the tag in a comment so Dependabot can propose
updates:

```yaml
- uses: xinp-hub/repo-policy/classify@<40-character commit SHA> # v1.0.0
```

A tag can be moved; a SHA cannot. The caller stays responsible for everything that makes the
decision binding, and none of it lives here:

- run the merge policy under `pull_request_target`, so the workflow file and the SHA it pins
  are the base branch's, and check the pull request out as data that is never installed,
  imported or run;
- keep the merge App's key in an environment that only the default branch can use;
- classify every change to `.github/` (and therefore every bump of these pins) as Tier 2,
  which `merge_policy.py` does: Tier 1 is an allowlist of `src/`, `tests/`, `docs/`,
  `README.md` and `CHANGELOG.md`, and Tier 0 is only `uv.lock` and `pyproject.toml`.

### classify

```yaml
- name: Classify
  id: tier
  uses: xinp-hub/repo-policy/classify@<sha> # vX.Y.Z
  with:
    working-directory: head
    base: ${{ github.sha }}
    head: ${{ github.event.pull_request.head.sha }}
    commits-json: ${{ runner.temp }}/commits.json
    pushers-json: ${{ runner.temp }}/pushers.json
    deps-json: ${{ runner.temp }}/deps.json
    api-break: ${{ steps.api.outputs.break }}
    authors: ${{ vars.AUTOMERGE_AUTHORS }}
```

### review

The reviewer commands and keys are passed as the step's environment, never as inputs, and
`agent_review.py` gives each reviewer only its own key, a minimal environment and an empty
working directory. A command reaches the bundled tool-less reviewer through
`$AGENT_REVIEW_SCRIPTS`, the absolute path of this repository's `scripts/` at the pinned
commit:

```text
AGENT_REVIEW_COMMAND = python3 "$AGENT_REVIEW_SCRIPTS/gemini_review.py" --model <id> [--model <id>]
```

```yaml
- name: Review
  id: review
  uses: xinp-hub/repo-policy/review@<sha> # vX.Y.Z
  with:
    working-directory: head
    base: ${{ github.sha }}
    head: ${{ github.event.pull_request.head.sha }}
    rubric: base/.github/agent-review.md
  env:
    AGENT_REVIEW_COMMAND: ${{ vars.AGENT_REVIEW_COMMAND }}
    AGENT_REVIEW_KEY_ENV: ${{ vars.AGENT_REVIEW_KEY_ENV }}
    AGENT_REVIEW_API_KEY: ${{ secrets.AGENT_REVIEW_API_KEY }}
```

The rubric is the calling repository's own file, read from its base branch.

### attribution

```yaml
- uses: actions/checkout@<sha> # v7
  with:
    fetch-depth: 0
    persist-credentials: false
- uses: xinp-hub/repo-policy/attribution@<sha> # vX.Y.Z
  with:
    base: ${{ github.event.pull_request.base.sha }}
    head: ${{ github.event.pull_request.head.sha }}
```

## License

MIT. See [LICENSE](LICENSE).
