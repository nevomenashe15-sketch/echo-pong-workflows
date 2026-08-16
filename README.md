# echo-pong-workflows

Reusable GitHub Actions workflows for the `echo-pong` application and its
sibling infrastructure/GitOps repositories. Every workflow here is
`on: workflow_call` only — nothing in this repository triggers itself. Callers
own their triggers, their protection rules and their credentials; this
repository owns the CI/CD *logic*.

> **Status: local staging copy for review.** Not initialised as a Git
> repository, not pushed, never executed against real GitHub Actions or AWS.

## Repository layout

```
echo-pong-workflows/
├── README.md
├── .yamllint.yml
├── .github/workflows/
│   ├── go-ci.yml                     # lint / vet / build / test  (no cloud access)
│   ├── image-build-quarantine.yml    # multi-arch build -> quarantine ECR (+ GHCR)
│   ├── image-scan.yml                # per-architecture Trivy scan + Syft SBOM
│   ├── image-promote.yml             # crane copy -> production ECR, Cosign, provenance
│   ├── gitops-propose-update.yml     # opens a PR against echo-pong-gitops (never kubectl)
│   ├── terraform-validate.yml        # fmt/validate/TFLint/Checkov/Trivy/Gitleaks + opt-in plan
│   ├── helm-validate.yml             # yamllint / helm lint / helm template / kubeconform
│   └── release.yml                   # cross-compiled binaries -> GitHub Release
└── examples/
    └── echo-pong-caller-workflow.yml # ILLUSTRATIVE thin caller (not executed)
```

## The pipeline these compose into

```
go-ci  ──▶  image-build-quarantine  ──▶  image-scan  ──▶  [human approval gate]
                    │                         │              (GitHub Environment,
                    │                         │               required reviewers)
                    ▼                         ▼                     │
              echo-pong-quarantine    fail closed on                ▼
              (ECR) + GHCR            HIGH/CRITICAL          image-promote
                                      per architecture    (crane copy, no rebuild)
                                                            Cosign + provenance
                                                                    │
                                                                    ▼
                                                        gitops-propose-update
                                                          (PR only — Argo CD
                                                           syncs after merge)
```

The two ECR repositories are not two names for one thing. **Pushed to a
registry ≠ approved for production.** `echo-pong-quarantine` holds untrusted
build output on a short lifecycle; `echo-pong` holds only digests that passed
scanning and were copied across verbatim. Scanning an image that is already in
the production repository does not prevent its publication — it only tells you
what you already shipped.

## Registry strategy

Both paths are supported deliberately, and they are not in tension:

| Path | Registry | Purpose |
| --- | --- | --- |
| Base deliverable | GHCR (`ghcr.io/<owner>/echo-pong`) | The app repo's README requires images in GitHub Container Registry. |
| AWS production | ECR `echo-pong-quarantine` → `echo-pong` | What the EKS cluster actually pulls, via the quarantine/promote flow. |

`image-build-quarantine.yml` pushes **one manifest list to both registries in a
single buildx invocation**, so GHCR and quarantine hold byte-identical content
under the same digest rather than two independently built images. Only the ECR
copy is subject to promotion; GHCR is a distribution mirror, not an approval
state.

## Contract reference

### Inputs / outputs / permissions

| Workflow | Required inputs | Optional inputs (default) | Secrets | Outputs | Job permissions | AWS role |
| --- | --- | --- | --- | --- | --- | --- |
| `go-ci.yml` | — | `go-version` (`1.24`), `working-directory` (`.`), `golangci-lint-version` (`v2.1.6`), `golangci-lint-timeout` (`5m`), `go-cache` (`false`), `test-flags` (`-race -cover -covermode=atomic`), `runs-on` (`ubuntu-latest`) | — | `coverage-percent` | `contents: read` only | **none** |
| `image-build-quarantine.yml` | `aws-region`, `aws-role-to-assume` | `ecr-quarantine-repository` (`echo-pong-quarantine`), `dockerfile` (`Dockerfile`), `context` (`.`), `platforms` (`linux/amd64,linux/arm64`), `image-tag` (commit SHA), `push-ghcr` (`true`), `ghcr-image-name` (`ghcr.io/<caller>`), `build-args` (`""`), `runs-on` | — | `image-digest`, `quarantine-image-ref`, `quarantine-repository-uri`, `ghcr-image-ref` | job `build`: `contents: read`, `id-token: write`, `packages: write` | `echo-pong-gh-ecr-release` |
| `image-scan.yml` | `aws-region`, `aws-role-to-assume`, `image-digest` | `ecr-quarantine-repository`, `severity` (`HIGH,CRITICAL`), `ignore-unfixed` (`false`), `sbom-format` (`cyclonedx-json`), `artifact-retention-days` (`30`), `trivy-timeout` (`10m`), `runs-on` | — | `scanned-digest`, `platforms-scanned`, `sbom-artifact` | jobs `enumerate`/`scan`: `contents: read`, `id-token: write`; `summarise`: `contents: read` | `echo-pong-gh-ecr-release` |
| `image-promote.yml` | `aws-region`, `aws-role-to-assume`, `image-digest` | `source-repository` (`echo-pong-quarantine`), `target-repository` (`echo-pong`), `production-tag` (`""`), `sign` (`true`), `attest-provenance` (`true`), `runs-on` | — | `production-digest`, `production-image-ref`, `production-repository-uri`, `cosign-identity` | job `promote`: `contents: read`, `id-token: write`, `attestations: write` | `echo-pong-gh-ecr-release` |
| `gitops-propose-update.yml` | `gitops-repository`, `environment`, `image-repository`, `image-digest` | `chart-path` (`charts/echo-pong`), `values-file` (derived), `base-branch` (`main`), `use-github-app` (`true`), `pr-labels`, `draft` (`false`), `runs-on` | `gitops-app-id`, `gitops-app-private-key`, `gitops-token` (all optional; one pair required) | `pull-request-url`, `pull-request-number` | job `propose`: `contents: read` only | **none** |
| `terraform-validate.yml` | `chart`/none | `terraform-version` (`1.9.8`), `working-directory` (`.`), `fmt-check-recursive` (`true`), `tflint-version` (`v0.53.0`), `checkov-soft-fail` (`false`), `checkov-skip-checks` (`""`), `trivy-config-severity` (`HIGH,CRITICAL`), `run-gitleaks` (`true`), `run-terraform-docs-check` (`true`), `terraform-docs-working-dir` (`.`), `run-plan` (`false`), `plan-role-to-assume` (`""`), `aws-region` (`""`), `plan-args` (`""`), `runs-on` | — | `plan-exitcode` | jobs `validate`/`lint`/`security`/`docs`: `contents: read`; job `plan`: `+ id-token: write` | `echo-pong-gh-tf-plan` (only when `run-plan: true`) |
| `helm-validate.yml` | `chart-path` | `environments` (`["dev","staging","production"]`), `values-file-prefix` (`values-`), `values-file-suffix` (`.yaml`), `helm-version` (`v3.16.3`), `kubeconform-version` (`0.6.7`), `kubernetes-version` (`1.31.0`), `yamllint-config` (`""`), `yamllint-paths` (`.`), `release-name` (`echo-pong`), `namespace` (`echo-pong`), `strict-lint` (`true`), `runs-on` | — | — | all jobs: `contents: read` | **none** |
| `release.yml` | — | `go-version` (`1.24`), `binary-name` (`ping-pong-game`), `working-directory` (`.`), `platforms` (`["linux/amd64","linux/arm64","darwin/arm64"]`), `version` (tag), `sign-checksums` (`true`), `draft` (`false`), `generate-release-notes` (`true`), `runs-on` | — | `release-url`, `version` | job `build`: `contents: read`; job `publish`: `contents: write`, `id-token: write` | **none** |

`terraform-validate.yml` has no required inputs; every input is defaulted.

### IAM roles referenced

Provisioned by `echo-pong-infrastructure`, referenced here **only** as
caller-supplied input values. No account ID, ECR hostname or role ARN is
hardcoded anywhere in this repository.

| Role | Used by | Notes |
| --- | --- | --- |
| `echo-pong-gh-ci-validation` | *nobody, by design* | Read-only. Available if a validation step ever needs AWS describe calls; `go-ci.yml` needs none. |
| `echo-pong-gh-ecr-release` | `image-build-quarantine.yml`, `image-scan.yml`, `image-promote.yml` | Push to quarantine, pull for scanning, copy to production. |
| `echo-pong-gh-tf-plan` | `terraform-validate.yml` (`run-plan: true` only) | Opt-in per call. |
| `echo-pong-gh-tf-apply` | **not used here** | Belongs to `echo-pong-infrastructure`'s own apply workflow, gated by the `production-infra` Environment. |

### Caller responsibilities

A reusable workflow's jobs are capped by the **calling job's** `permissions:`
block. Grants that are missing there cause hard failures, not silent skips.
Callers must therefore declare the permissions listed above themselves, plus:

- `production-promotion` GitHub Environment with required reviewers, if the
  promotion gate is wanted (it is not enforced by this repository).
- Repository variables for `aws-region` and role ARNs.
- A GitHub App or fine-grained PAT scoped to the GitOps repository.

## Versioning

Consumers **must not pin `@main`.** A reusable workflow referenced by a branch
is re-resolved on every run, so any commit here silently changes every
consumer's pipeline at once, with no review on their side and no rollback other
than reverting this repository.

Tagging strategy for this repository once it has a release:

- Immutable `vMAJOR.MINOR.PATCH` tags (`v1.2.3`) for every release.
- A floating `vMAJOR` tag (`v1`) that consumers pin to, moved forward on each
  backward-compatible release. This is standard GitHub Actions practice: it
  gives consumers fixes without breaking changes.
- Breaking input/output/permission changes bump the major and require consumers
  to move to `@v2` deliberately.
- For a fully audited supply chain, pin to a commit SHA and bump via Dependabot.

The example caller uses `@v1` throughout. This is the one convention the
precedent repository (`ci-cd-templates`) explicitly flagged as wrong in its own
README, and it is not repeated here.

## Design decisions

**arm64 via QEMU, not native runners.** GitHub-hosted `ubuntu-latest` is
amd64-only. QEMU emulation is nearly free for this app specifically: the
Dockerfile pins its build stage to `--platform=$BUILDPLATFORM` and cross-compiles
with `GOOS`/`GOARCH`, so the Go compile always runs natively and only the
final-stage `COPY` into distroless is emulated. Native arm64 runners are a
future build-speed optimisation, not a present need.

**Trivy over Grype.** One scanner across the organisation (`ci-cd-templates`
already uses Trivy) means one severity vocabulary and one ignore mechanism.
Trivy also reads Go binary build info directly, which matters because the
distroless final stage has no package manager to interrogate. Grype would work;
running both doubles triage cost without doubling coverage.

**Per-architecture scanning.** A manifest list contains no layers. Pointing a
scanner at the list makes it resolve one child — the runner's architecture — so
the arm64 image would never be scanned. `image-scan.yml` enumerates the index
and fans out over every real platform child, filtering the `unknown/unknown`
attestation manifests buildx adds.

**Syft, CycloneDX JSON, per architecture.** Syft is the generator behind Trivy's
own SBOM output. CycloneDX was chosen over SPDX for its first-class
VEX/vulnerability linkage. Per-architecture documents for the same reason the
scan fans out: the children have different contents.

**Promotion by `crane copy`.** Builds are not bit-for-bit reproducible, so
rebuilding after a passing scan yields a different digest and the scan no longer
describes what shipped. `crane copy` transfers the index, children and blobs
verbatim; `image-promote.yml` then asserts the destination digest equals the
scanned digest and fails if not. There is no `docker build` in that workflow.

**GitHub App over PAT for cross-repo PRs.** The ambient `GITHUB_TOKEN` cannot
write to another repository. An App installed on `echo-pong-gitops` alone mints
one-hour tokens, is not tied to an employee account, and cannot be widened
without an audited change. A fine-grained PAT scoped to that one repository is
an acceptable interim. A classic `repo`-scope PAT is not acceptable for either.

**Binary matrix.** `linux/amd64` + `linux/arm64` mirror the container targets
and the cluster's Graviton preference; `darwin/arm64` covers operators on Apple
Silicon. Windows and `darwin/amd64` are omitted — no stated consumer, and each
extra target is another artefact to sign and support. Overridable via input.

**Checksums signed, individual binaries not.** The checksum manifest already
commits to every artefact's content, so one Cosign signature covers the whole
release and verification stays a two-step check.

## Known risks and follow-ups

- **Cosign identity names *this* repository, not the caller.** Signing happens
  inside `image-promote.yml`, so Fulcio issues the certificate against the
  `job_workflow_ref` claim. Admission policy must expect
  `.../echo-pong-workflows/.github/workflows/image-promote.yml@<ref>`. The
  workflow prints the identity Cosign actually recorded so the policy can be
  copied from a real run.
- **ECR tag immutability vs Cosign signature tags.** Cosign writes a
  `sha256-<digest>.sig` tag. With tag immutability enabled on `echo-pong`, a
  re-sign of an already-signed digest will fail. Verify against a real registry
  before release; use OCI 1.1 referrers if it bites.
- **Action version pins are unverified offline.** Third-party action tags
  (notably `aquasecurity/trivy-action`, `imjasonh/setup-crane`) could not be
  resolved without network access to the registry. Verify — and ideally
  SHA-pin — before tagging `v1`.
- **`yq -i` normalises blank lines** between top-level keys in the GitOps values
  file, making the PR diff slightly noisier than the one-line change. Cosmetic.
