# bulk-pr

**bulk-pr** is a **scheme** for performing operations such as creating Pull Requests (PRs) across multiple repositories within a GitHub Organization.
By implementing your GitHub Actions workflows according to the bulk-pr scheme, you can perform batch operations on repositories in your organization.

bulk-pr is designed for medium-to-large development organizations.
For individuals or small teams, [multi-gitter](https://github.com/lindell/multi-gitter) is more suitable.

## :warning: Status

This project is still **work in progress**.
It’s **not production ready** yet.
The documentation or sample workflows may contain errors.

## Features

* **Established scheme, documentation, and sample code**

  * You can implement it easily by following the provided documentation and samples instead of designing everything from scratch.
* **Batch operations via GitHub Actions**

  * Environment-independent
  * Enables pre-review before execution
  * No need to grant individuals access to sensitive Secrets
* **Centralized workflows in a dedicated GitHub repository**

  * For convenience, this repository is called the `bulk-pr` repository
  * Ensures security and governance
  * Accumulates operational knowledge in one place, making it easier to reference and reuse existing workflows
* **PR creation via GitHub App**

  * Not tied to a specific individual
* **Secure PR creation with Securefix Action**

  * No need to pass PR creation secrets to task workflows, ensuring safety
* **PR management via GitHub Projects**

  * Each task can have its own dedicated GitHub Project

## Setup

* Set up **Securefix Action**

  * Allow PR creation from the default branch of the `bulk-pr` repo
  * Grant **organization projects: write** permission
* Create the `bulk-pr` repository
* Create a **GitHub App**

  * Webhook: disabled
  * Permissions:

    * contents: read
    * pull requests: read
    * organization projects: read
  * Installed repositories: `bulk-pr` and the target repositories to be modified

## Repository Structure

```
.github/workflows/
  workflow-1.yaml  # workflow_dispatch to create PRs in bulk
tasks/
  workflow-1/      # (optional) files required by workflow-1
```

Each batch PR operation is called a **task**.
Create one workflow per task under `.github/workflows/<task name>.yaml`.
If the task requires scripts, place them in `tasks/<task name>/`.

The `tasks/<task name>/` directory is optional — you may also fetch scripts from external repositories within the workflow.
However, if you do so, be careful about **security risks** (e.g., tampered external scripts).

## Adding a Task

1. Create a dedicated GitHub Project
2. Open a PR to add the workflow to the `bulk-pr` repository
3. Review and merge it into the default branch
4. Test the workflow with test data; fix as needed
5. Run the workflow
6. Once the task is completed, move the workflow file from `.github/workflows/<task name>.yaml` to `tasks/<task name>/workflow.yaml`

   * Leaving unused workflows under `.github/workflows` can clutter the directory
   * You can delete them, but keeping them in `tasks/<task name>` helps as references for similar future tasks

## Example

* [Example of pinning versions with pinact](.github/workflows/all-repos.yaml)

## Workflow Structure

See [Example](.github/workflows/all-repos.yaml)

Workflows are implemented with `workflow_dispatch`.
However, due to constraints of OIDC and GitHub Environment Secrets, they are configured to run only on the **default branch**.

Each workflow consists of two jobs:

* **list** — generates a list of PR targets (passed to the matrix job)
* **create-pr** — modifies code and creates PRs

## list Job

The `list` job generates the list of repositories or files where PRs will be created.
It usually uses the GitHub API, or shell commands like `find` or `grep`.

Accessing other private repositories via the GitHub API requires a GitHub App.

```sh
{
  echo "value<<EOF"
  gh repo list szksh-lab -L 10000 --json nameWithOwner
  echo "EOF"
} >> "$GITHUB_OUTPUT"
```

Workflows must be **idempotent** — already-created PRs must be excluded from the list.
bulk-pr ensures this by using a dedicated GitHub Project per task and skipping repositories that already have PRs listed in the project.
If you need to recreate a PR, remove it from the project first.

Since matrix jobs can run only up to **256 jobs**, only 256 PRs are processed per run.
If there are more, execute the workflow multiple times.

## create-pr Job

The `create-pr` job performs:

1. Checkout the target repository
2. Modify the code
3. Create a PR using Securefix Action

If your code modification needs additional scripts, add them to the `bulk-pr` repo and checkout both repositories.
Be careful to avoid path conflicts.

When using Securefix Action, PR creation is **asynchronous**, so even if the workflow succeeds, PRs may not have been created yet.

## Testing Workflows

Running a workflow that creates many PRs at once without testing is risky.
Design workflows to accept test data via `workflow_dispatch` and create only a single PR for verification.
Once the PR title, description, and changes are confirmed, you can safely execute the workflow for all repositories.

## Searching Repositories, Code, or PRs via GitHub API

If you just need to list repositories, `gh repo list` is convenient:

```sh
{
  echo "value<<EOF"
  gh repo list szksh-lab -L 10000 --json nameWithOwner
  echo "EOF"
} >> "$GITHUB_OUTPUT"
```

Note that filtering options like `--no-archived` or `--source` limit results to **1,000 repositories** due to GitHub API restrictions.

```sh
gh repo list szksh-lab -L 1000 --json nameWithOwner --no-archived --source
```

Search API queries also return at most **1,000 results**:
[https://docs.github.com/en/graphql/reference/queries#search](https://docs.github.com/en/graphql/reference/queries#search)

To use more flexible queries, try `gh search repos`.
To search for code, use [gh search code](https://cli.github.com/manual/gh_search_code):

```sh
gh search code "" --filename .goreleaser.yml --owner suzuki-shunsuke -L 1000 --json path,repository
```

Example output:

```json
[
  {
    "path": ".goreleaser.yml",
    "repository": {
      "id": "R_kgDOMOw9Zg",
      "isFork": false,
      "isPrivate": false,
      "nameWithOwner": "suzuki-shunsuke/ghatm",
      "url": "https://github.com/suzuki-shunsuke/ghatm"
    }
  }
]
```

You can process the results with `jq`:

```sh
gh search code "" --filename .goreleaser.yml --owner suzuki-shunsuke -L 1000 --json path,repository --jq "[.[] | {path, nameWithOwner: .repository.nameWithOwner}]"
```

To search PRs within a GitHub Project, use the GraphQL API:

```sh
gh api graphql --paginate -f query='
query {
  search(first: 100, query:"is:pr project:szksh-lab/1", type:ISSUE) {
    pageInfo {
      hasNextPage
      endCursor
    }
    nodes {
      ... on PullRequest {
        repository {
          nameWithOwner
        }
        number
        headRefName
      }
    }
  }
}
'
```

<details>
<summary>Notes</summary>

* `gh search prs` is convenient for finding PRs, but it doesn’t return branch names.
* Using `projectV2` in GraphQL to fetch all items retrieves issues as well, making it inefficient.

</details>

## Security Considerations

While bulk-pr enables convenient batch PR creation, it must not be exploitable.
If abused, a malicious actor could generate and self-approve harmful PRs.

Always carefully review PRs to the `bulk-pr` repository to ensure no malicious code or external dependency can be tampered with.
For example, if an external script is executed, it could be modified later to run arbitrary commands.
To prevent that, always pin scripts by full commit SHA, verify checksums, or manage them in **protected branches**.

## Batch Operations Other Than PR Creation

Other batch operations you might want include:

* Closing PRs
* Merging PRs
* Approving PRs
* Commenting on PRs
* Updating PRs (labels, reviewers, assignees, title, description, etc.)
* Removing PRs from Projects
* Adding commits to PRs

For closing or merging PRs, a simple local script is usually sufficient.
Running them via GitHub Actions provides benefits like reviewability and environment consistency, but the advantages are smaller than for PR creation.
PR creation often requires repository checkout and script execution, where GitHub Actions truly shines.
For closing/merging, local execution via GitHub CLI is typically enough.

Of course, if you prefer running them through GitHub Actions, you can implement workflows similar to PR creation.

## Why bulk-pr?

### Compared to Running Local Scripts

The simplest way to create PRs in bulk is to write a local script using your personal access token.
However, this approach causes problems in large organizations:

* Excessive PR notifications
* Reviewers may expect the PR author (you) to handle the PR, making ownership transfer difficult

Since PRs are authored by you, all notifications (reviews, merges, etc.) will go to your account — very noisy, especially if integrated with Slack.
Also, managing many PRs yourself is impractical, especially when repository ownership is distributed.

Ideally, PRs should be created in bulk but handled individually by repository owners.
However, if the PR author is a person, reviewers may naturally assume that person is responsible.

bulk-pr solves this by using a **GitHub App** (or optionally a Machine User) instead of a personal access token.
This eliminates personal notifications and makes ownership delegation clearer.

While it’s technically possible to use a GitHub App or Machine User token locally, that would require distributing secrets to individuals — a security risk.
bulk-pr runs within GitHub Actions, so no secrets need to be shared manually, and all execution is auditable.

### PR Creation with Securefix Action

bulk-pr uses [Securefix Action](https://github.com/csm-actions/securefix-action) for creating PRs.
Workflows that create PRs are managed separately from the Securefix Action Server repository.
This ensures the workflow itself doesn’t have direct access to sensitive PR-creation secrets.

Allowing arbitrary repositories to freely create PRs with GitHub App tokens poses risks,
but Securefix Action mitigates this by restricting allowed repositories and branches, significantly reducing the attack surface.

## LICENSE

[MIT](LICENSE)
