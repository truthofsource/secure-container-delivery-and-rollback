# DevSecOps Delivery & Verified Rollback

**Release the tested files and keep a verified previous version ready if something goes wrong.**

[The problem](#the-business-problem) · [The solution](#the-solution) · [What was built](#what-was-built) · [Proof it works](#proof-it-works) · [Run it](#run-it-locally)

![Overview of the working demonstration](docs/images/01-overview.png)

## The Business Problem

A green build did not show exactly which application package or container would reach production. Rebuilding during release could change the files, and an old branch was not a dependable recovery package. Security reports also needed to match the version being released.

## The Solution

Build once, keep the tested package and its fingerprint, and check the matching security results before release. Save a verified previous package for rollback, which means returning to an earlier release. Keep permission to publish packages separate from the application’s own access.

**Business outcome:** Make releases traceable and recovery deliberate, with fewer differences between the tested package and the released one.

### Platform implementation

- Built API/web saved release packages, exact-hash readiness checks and paired promotion so production used the verified packages instead of a new build.
- Added release and rollback checks for exact selected source, retained artifacts, settings preservation and cleanup of temporary release infrastructure.
- Integrated dependency, container, IaC and secret scanning with dated findings, package inventories and remediation evidence; separated report generation from release authorization.
- Patched vulnerable dependencies using compatible versions and tests that existing callers still work, retaining unresolved findings rather than claiming a clean estate.

This case study draws on a workforce/operations platform I designed and implemented. Organization details and records are excluded. The public code makes selected patterns reproducible; the screenshots show a working local demonstration with fictional data. [Validation scope](docs/VALIDATION.md) separates that evidence from testing against the actual cloud service.

### Actual implementation code

This repository now includes selected implementation source and original tests from the operational platform, rather than only a portable reference. Read [what was implemented and accepted](docs/IMPLEMENTATION.md), [the source map](docs/SOURCE-MAP.md) and the dated validation boundaries. Identifying configuration is replaced with examples; original private history and records are excluded.

## Architecture

```mermaid
flowchart LR
  Source[Reviewed commit] --> Build[Build and test]
  Build --> Scan[Dependency / image / IaC / secret scans]
  Scan --> Gate[Trusted policy gate]
  Build --> Archive[(Retained ZIP / digest)]
  Gate --> Select[Exact source and artifact selection]
  Archive --> Select
  Select --> NP[Nonproduction verification]
  NP --> Prod[Same bytes to production]
  Archive --> Rollback[Retained previous release]
```

## What Was Built

| Component | Role in plain words | Implementation |
| --- | --- | --- |
| Actual rollback guards | Bind exact source/artifact, expiring candidate, compatibility and live-state drift | [Source](reference/rollback_release.py) · [original selected tests](reference/test_rollback_release.py) |
| Deterministic package builder | Sort paths and fix ZIP metadata so identical inputs produce identical bytes. | [View code](lab/domain.py) |
| Release-file checks | Reject changed files, reports for another release, unfinished scans and findings that block release. | [View code](lab/domain.py) |
| Content-addressed release archive | Retain packages and change a small atomic release pointer instead of rebuilding. | [View code](lab/domain.py) |
| ACR and repository-specific publisher | Keep registry administrator passwords Off and limit optional GitHub publishing access to one image repository. | [View code](infra/registry.tf) |
| Private artifact storage | Use keyless, versioned archive storage with deletion protection. | [View code](infra/storage.tf) |
| Container example | Run the example with a lower-permission account in an exact, verified Python container build. | [View code](Dockerfile) |

## Proof It Works

### Extracted platform code

```bash
python -B -m unittest discover -s reference -v
```

The selected original tests run offline. [Validation](docs/VALIDATION.md) records current counts and distinguishes them from historical hosted evidence.


**Current local checks:** 12 extracted-source tests + 12 portable tests passed · 6 demonstration checks · 2 Terraform tests using simulated cloud responses.

These are dated results for this reference implementation, not live Azure/AWS deployment claims. [Test record](docs/validation-results.json) · [Evidence and limitations](docs/VALIDATION.md)

### 1. Promotion preserves the selected bytes

The local test compares the promoted package against the original. Changed bytes and stale/incomplete evidence fail before publication.

Run from the repository root:

```bash
python -B -m unittest discover -s tests -p test_domain.py -v
```

**What to look for:** The package tests reject changed bytes, wrong source, out-of-date evidence and incomplete/blocked scans.

### 2. Rollback uses the retained prior package

The demonstration re-verifies a previous archive and selects its digest. It does not rebuild the old source.

Run from the repository root:

```bash
python -B -m lab demo
```

**What to look for:** The JSON records the current and retained previous package digests and the restored selection.

### 3. The container runs under restrictions

The recorded Linux ARM64 smoke used a non-root user, no network, a read-only filesystem and dropped capabilities. This was a runtime test, not a vulnerability scan of that demonstration image.

Run from the repository root:

```bash
docker build -t delivery-reference:local .
docker run --rm --network none --read-only --cap-drop ALL --security-opt no-new-privileges --tmpfs /tmp:rw,noexec,nosuid,size=16m delivery-reference:local demo
```

**What to look for:** This reproduces the recorded Linux container smoke using a portable local tag. The run is non-root and network-isolated; it is not a vulnerability scan.

### 4. Visible results

The dashboard runs the same domain code as the CLI. Desktop/mobile rendering, all three tabs, rerunning the demonstration, cross-origin write rejection and unlisted-file rejection were checked. The screenshots are captured from those runs.

![Positive and negative checks from the running demonstration](docs/images/02-validation.png)

Expected access denials are successful negative tests. Cloud permissions, network connectivity and provider integrations need their own acceptance evidence.

## Security Practices

- The local policy receipt is test-only. A real release needs independently trusted scanner evidence and source-to-artifact binding.
- ACR admin credentials and storage shared keys are Off. The optional publisher uses an exact reviewed OIDC subject and repository ABAC condition.
- Trivy, Checkov, TruffleHog and ZAP have different coverage. Remaining findings stay visible; a digest or green build alone does not establish safety.

State, saved Terraform plans, private inputs and credentials are excluded through [`.gitignore`](.gitignore). The existing validation workflow is manual-only and has no cloud deployment permission.

## Run It Locally

The [command reference](docs/COMMANDS.md) includes prerequisites, Windows equivalents, focused checks and evidence notes.

Requires Python 3.12 or newer. The default demonstration uses the Python standard library. Run inside a virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -B -m unittest discover -s tests -v
python -B -m lab demo
python -B -m lab serve --port 8011
```

On Windows, use `.venv\Scripts\Activate.ps1` instead of the `source` command, or run `.venv\Scripts\python.exe` directly. Open [the local workbench](http://127.0.0.1:8011), choose **Run demonstration**, then review **Validation** and **Evidence**. Stop with Ctrl+C. No cloud login, employee data or paid API request is required.

## Terraform Infrastructure

A shared-style ACR registry, keyless private release-archive storage with version protection, and an opt-in repository-specific OIDC publisher.

See the [Terraform code and validation steps](infra/README.md), [input example](infra/example.tfvars.example) and [mock plan tests](infra/tests). Cloud commissioning and any paid resources remain separate from the local demonstration.

Validate the infrastructure source without creating cloud resources:

```bash
cd infra
terraform init -backend=false
terraform fmt -check -recursive
terraform validate
terraform test
```

The tests use mock providers. Example inputs are placeholders, not deployment-ready values. Review the [integration boundaries and cost notes](infra/README.md) before any separate cloud commissioning.

## Scaling and Tradeoffs

Separate build, scan, select and publish responsibilities. Retain verified release pairs and referenced images, use trusted signing/provenance checks and avoid repeating successful jobs when only one stage fails.

See [strengths, weaknesses and tradeoffs](docs/SCALING.md) for the detailed decisions.

## Skills Demonstrated

- GitHub Actions and artifact promotion
- Container packaging and runtime restrictions
- Vulnerability/secret/IaC scan integration
- OIDC and repository-specific publishing
- Verified rollback and release traceability

## Project Guides

| Guide | What you will find |
| --- | --- |
| [Command reference](docs/COMMANDS.md) | Copy the commands, understand their purpose and check expected results |
| [Implementation walkthrough](docs/WALKTHROUGH.md) | Follow the processing path and code |
| [Architecture](docs/ARCHITECTURE.md) | Understand service and identity boundaries |
| [Terraform](infra/README.md) | Read the resource map, input examples and integration steps |
| [Validation evidence](docs/VALIDATION.md) | See the test dates, results and limits |
| [Operations and troubleshooting](docs/OPERATIONS.md) | Diagnose failures and recover safely |
| [Security and privacy](SECURITY.md) | Review access, secrets and publication controls |
| [Scaling and tradeoffs](docs/SCALING.md) | Understand the costs of each design choice |
| [References](docs/REFERENCES.md) | See technical sources and presentation inspiration |

## Repository Map

`reference/` contains extracted implementation logic and original selected tests.


```text
infra/               Terraform source, example inputs and mock plan tests
lab/                 Working reference implementation and loopback dashboard
tests/               Positive, negative and recovery cases
docs/                Architecture, walkthrough, evidence and operating guide
.github/workflows/   Manual-only contract validation; no cloud deployment
```

## Cost and Cleanup

The default local demo creates only temporary local state and makes no paid calls. Cloud examples are separate opt-in source and can create billable resources if an operator applies them. No automated deployment or destruction is configured. Use `git status --short --ignored` before cleanup; remove only your own `.venv` or disposable lab output. Never copy real state files, tokens or employee documents into this repository.
