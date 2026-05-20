# deployment-automation-service
FastAPI orchestration service for automated multi-stack project deployment (Gitea + Jenkins + AWS Lambda)

# Deployment Automation Service

A FastAPI orchestration service that automates end-to-end project deployment across Git, CI, and cloud targets. Built as an internal engineering productivity tool — engineers go from "code pushed" to "service running on a live URL" without manually touching Git hosts, CI servers, or cloud consoles.

> **Note on this repository:** Source code is not public because the implementation was built as an internal tool at my employer. This README documents the architecture, the components I built, and the design decisions I made — both to share what I learned and to give context for technical interviews. Happy to walk through the code in person.

---

## What it does

Given a project (a Python service, a FastAPI app, or an npm-based frontend), the service:

1. Pushes or updates the project's source in a Gitea repository (creating the repo and branch if needed)
2. Generates a `Jenkinsfile` tailored to the project's tech stack and commits it to the repo
3. Creates or updates a Jenkins job pointing at that repo and branch
4. Triggers the build and streams the logs back to the caller
5. (For Lambda targets) builds a container image, pushes it to ECR, deploys it as a Lambda function, and provisions API Gateway routes
6. Persists a record of every deployment (project, build number, status, app URL) in PostgreSQL

The point: developers don't need to know Jenkins XML, Gitea's API, or how to configure systemd. They hit one endpoint and get a running service back.

---

## Architecture

```
                      ┌─────────────────────────┐
                      │   FastAPI Orchestrator  │
                      │  (this service's core)  │
                      └────────────┬────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
        ▼                          ▼                          ▼
   ┌─────────┐               ┌──────────┐              ┌─────────────┐
   │  Gitea  │               │ Jenkins  │              │ AWS Lambda  │
   │ (REST)  │               │  (REST)  │              │ (boto3)     │
   └─────────┘               └──────────┘              └─────────────┘
        │                          │                          │
        ▼                          ▼                          ▼
   repos, branches,         dynamic jobs,             ECR images,
   commits, ZIPs            triggered builds          API Gateway routes
                                   │
                                   ▼
                          ┌─────────────────┐
                          │   PostgreSQL    │
                          │ (build history) │
                          └─────────────────┘
```

---

## My contribution

I want to be specific here, because not everything in this service is mine. This was a team project; my work focused on the **Gitea integration layer** and the **FastAPI orchestration endpoints** that sit on top of it.

**What I designed and built:**

- The full **Gitea integration module** — wrapping Gitea's REST API to handle repository creation, branch creation, file commits (with SHA-based create-or-update logic), branch listing, and ZIP-archive uploads to existing or new repos
- The **FastAPI orchestration endpoints** that expose this functionality (six upload variants covering repo × branch existence permutations)
- The **error handling and validation logic** around Gitea operations — retry loops for the race condition where Gitea creates a repo asynchronously and the default branch isn't immediately available, conflict detection when branches already exist
- The **deployment-record persistence layer** — SQLAlchemy ORM models for build history, query helpers for retrieving the latest build per project per user

**What I contributed to but didn't design:**

- The Jenkins integration (job XML generation, build triggering, log streaming) — I worked with this code but the structure came from a teammate
- The AWS Lambda + ECR deployment flow — same: I understood it and integrated against it, but the architecture wasn't mine
- The dynamic Jenkinsfile generation (per-stack templates for Python/FastAPI/npm) — collaborative work

I'm calling this out because in an interview I want to be able to go deep on the parts I actually built, and be honest about the parts where I'd be paraphrasing my teammates' design decisions.

---

## Design decisions I made (and why)

### 1. SHA-based create-or-update for file commits

Gitea's contents API requires the existing file's SHA when updating a file (`PUT`), but rejects the SHA when creating a new one (`POST`). My commit function checks for an existing SHA first, then chooses the verb accordingly. Without this, you'd get either a 404 on first commit or a 409 conflict on updates — and the caller would have to track per-file state, which they shouldn't have to.

### 2. Six upload endpoints instead of one

There's a permutation of three states: repo exists/doesn't exist, branch exists/doesn't exist, and upload format is files-or-ZIP. I made each combination its own explicit endpoint rather than overloading a single one with flags. The reasoning: ambiguous endpoints with conditional behaviour are harder to test, harder to read, and tend to grow weird edge cases. Explicit endpoints made the API surface boring and predictable.

In hindsight: this was probably one or two endpoints too many. A single `/upload` with three boolean flags would have been fine and more standard.

### 3. Retry loop on repo creation

Gitea creates repositories asynchronously — the API returns 200 before the default branch is fully usable. My code polls for branch availability (up to 40 seconds, 2-second intervals) before attempting to create child branches off it. I learned this the painful way when our first version intermittently failed with "branch not found" errors on freshly-created repos.

### 4. Separate database for build history (PostgreSQL) from vector store (ChromaDB elsewhere)

This came up because the same overall platform also runs RAG workloads. We kept relational deployment metadata in PostgreSQL (strong consistency, structured queries like "latest build per project per user") and vector data in ChromaDB (high-dimensional similarity search). The access patterns are fundamentally different — putting them in the same store would mean compromising one for the other.

---

## Tech stack

- **Language:** Python 3.10+
- **Framework:** FastAPI (async endpoints, automatic OpenAPI docs)
- **Persistence:** PostgreSQL via SQLAlchemy ORM
- **External integrations:** Gitea REST API, Jenkins REST API, AWS (boto3 — Lambda, ECR, API Gateway, STS)
- **Auth:** API tokens for Gitea, Basic Auth for Jenkins, IAM credentials for AWS
- **Deployment:** Run as a systemd service behind uvicorn

---

## What I'd do differently

Writing this out has been a useful exercise — a few things I'd change if I were building it again:

1. **Idempotency keys on the upload endpoints.** Right now, retrying a failed upload can leave the repo in a partial state. Idempotency keys would let the client retry safely.
2. **Async Jenkins polling instead of blocking waits.** The current `wait_for_build_completion` blocks for up to 10 minutes. A webhook-based or async-task-based approach would scale better when many deployments run in parallel.
3. **More aggressive use of FastAPI's dependency injection.** I passed Gitea credentials as imported globals; they should be dependencies. Easier to test, easier to mock.
4. **Structured logging instead of f-string log lines.** Worked fine for a single internal service, but would have been a pain to query at scale.

---

## What I learned

- **Most integration work is dealing with edge cases the API docs don't mention.** The Gitea-async-branch-creation issue isn't in their docs anywhere — you find it by hitting the bug in production.
- **Synchronous design choices come back to haunt you.** A blocking 10-minute wait is fine for one user; it's not fine for ten.
- **Error messages are a UX surface.** Returning `Gitea API Error: HTTP 422` to a developer caller is useless. I learned to surface enough context that a developer could self-diagnose without reading my code.

---

## About me

I'm Kedar Hardikar, an AI/Generative AI engineer based in Pune, India. I build production LLM systems — RAG pipelines, document intelligence, FastAPI backends — for enterprise clients. Open to AI/ML Engineer roles.

- **LinkedIn:** [linkedin.com/in/kedarhardikar](https://www.linkedin.com/in/kedarhardikar)
- **GitHub:** [github.com/kedarhardikar](https://github.com/kedarhardikar)
- **Email:** hkedar0@gmail.com
