# Repo Factory

Self-service GitHub repository creation, with policy checks and a human approval step. A user fills in an issue form. The system validates the request, opens a pull request for approval, and creates the repository once that pull request is merged. Each step is reported back on the issue.

**Stack:** GitHub Apps, Azure Functions (Python), Durable Functions, Azure Storage Queues, Key Vault, Terraform, GitHub Actions.

## How it works

```mermaid
flowchart TD
	U[User] --> I[Issue Form\nrepo-factory]
	I --> A[Orchestrator\nGitHub App on Azure Functions]
	A -->|reads rules| P[repo-factory-policies]
	P --> V{Valid?}
	V -->|No| F[Issue comment with the reasons]
	V -->|Yes| R[Pull request in\nrepo-factory-registry]
	R --> L[Issue comment with the PR link]
	R --> M{Merged?}
	M -->|Closed| D[Issue comment: not approved]
	M -->|Merged| C[Repository created in the organization]
	C --> S[Issue comment with the repository link]
```

1. **Request.** The user opens an issue with the repository name, description, owning team and visibility. The bot replies that the request is being validated.
2. **Validate.** The orchestrator checks the request against the policy file and rejects names that already exist. Violations are listed in a comment, and editing the issue triggers a new check.
3. **Approve.** A valid request becomes a pull request in the registry that adds `repositories/<name>.yml`. A reviewer merges it, and the merge is the approval.
4. **Create.** On merge, the orchestrator checks the record against the policy again, creates the repository, and comments the link on the issue.

## Repositories

| Repository | Role |
|---|---|
| `repo-factory` | Entry point: the Issue Form and this documentation |
| `repo-factory-policies` | The rules every request is checked against |
| `repo-factory-registry` | Approval record: one metadata file per repository |
| `repo-factory-app` | The orchestrator, an Azure Function App acting as a GitHub App |
| `repo-factory-infra` | Terraform for the Azure resources and the deploy workflow |

A registry record looks like this:

```yaml
name: payments-api
description: API for processing payments
visibility: private
owning_team: team-payments
default_branch: main
request_issue: 123
requester: octocat
```

## Design decisions

- **Approval through a pull request.** Review, audit trail and permissions come from GitHub itself, so there is no custom approval system to build.
- **Policy as a separate repository.** Rules change without redeploying the app, and the same rules are applied again at creation time.
- **Queue, then durable orchestration.** The webhook only verifies the signature and enqueues, so GitHub gets a fast answer. The orchestration is keyed by the webhook delivery ID, so repeated deliveries are not processed twice.
- **Safe to retry.** An existing registry pull request or repository is reused instead of duplicated, and activities retry before the requester is told that creation failed.
- **The requester is never left guessing.** Every outcome appears as a comment on their issue.
- **Abuse protection.** The intake is public, so the service accepts one request per day. The slot is claimed with an atomic table insert, and refused requests are told when to retry.
- **Secrets in Key Vault.** The GitHub App key and the webhook secret are Key Vault references, and Azure access from CI uses OpenID Connect with no stored credentials.

## Roadmap

- Auto-approve low-risk requests.
- Apply default labels and branch protection to new repositories.
- Grant the owning team access to the repository it owns.
