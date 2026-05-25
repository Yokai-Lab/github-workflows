# github-workflows

Reusable GitHub Actions workflows for Yokai Labs — CI, npm publishing, and
AWS/Pulumi deploys for Node projects. Referenced via `uses:` from a git tag
(there's nothing to `npm install`). They assume the conventions below; a project
that doesn't match them won't work out of the box. See [`docs/bootstrap.md`](docs/bootstrap.md)
for the one-time account/repo setup these workflows depend on.

## Conventions these workflows assume

- **Package manager:** npm (`npm ci`, npm lockfile) — not pnpm or yarn.
- **Node:** 24 by default (overridable via `node-version`).
- **CI quality gates (`ci-node`):** your `package.json` must define `lint`
  (eslint), `knip` (dead-code), and `build` scripts, and tests run under vitest.
  No `knip` script ⇒ CI fails. The formatting/lint/TS configs themselves come
  from the shared [`@yokailabs/*` config packages](https://github.com/Yokai-Lab/yokailabs-configs)
  (`prettier-config` today; `eslint-config` and `tsconfig` planned) — the
  workflow only runs the scripts, so it's agnostic to which configs you extend.
- **Cloud:** AWS, region `us-east-2` by default.
- **No static credentials, anywhere.** Auth is GitHub OIDC end to end — AWS via
  an assumable IAM role (`AWS_DEPLOY_ROLE_ARN`), npm via trusted publishing.
  **Prerequisite:** the GitHub OIDC identity provider must already exist in the
  AWS account, and the role's trust policy must allow this repo's subject.
- **IaC:** Pulumi with a **self-managed S3 state backend**
  (`PULUMI_BACKEND_URL=s3://…`) and a **passphrase-encrypted secrets provider**
  (`PULUMI_CONFIG_PASSPHRASE`) — i.e. not Pulumi Cloud, not KMS. The Pulumi
  program is itself an npm project (it gets `npm ci`).
- **GitHub Environments = Pulumi stack names.** The preview job runs in
  `environment: <stack-name>`, so create a GitHub Environment per stack (e.g.
  `beta`, `prod`) and scope that stack's vars/secrets to it.
- **Publishing:** scoped npm packages (`@yokailabs/*`) via Changesets, with
  provenance on.
- **Branch:** `main`.
- **Notifications:** Slack incoming webhook.

## Versioning

Pin callers to the **moving major tag** (`@v1`). Breaking changes bump the major
(`v2`); a dormant project pinned to `@v1` keeps working. Don't pin to `@main`.

```yaml
uses: Yokai-Lab/github-workflows/.github/workflows/ci-node.yml@v1
```

## Available workflows

| Workflow                   | Description                                                                      |
| -------------------------- | -------------------------------------------------------------------------------- |
| `ci-node.yml`              | format (prettier), lint (eslint), dead-code (knip), build, test (vitest)         |
| `pulumi-preview-node.yml`  | `pulumi preview` on a Node program, AWS via OIDC, PR comment                     |
| `npm-publish.yml`          | version + publish `@scope` packages via Changesets + npm OIDC Trusted Publishing |
| `notify-slack-failure.yml` | post a build/deploy failure to Slack                                             |

---

## Usage

### Node CI

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main]
  workflow_call:

jobs:
  ci:
    uses: Yokai-Lab/github-workflows/.github/workflows/ci-node.yml@v1
    with:
      node-version: "24" # optional, this is the default
```

Requires `lint`, `knip`, and `build` scripts in `package.json`, prettier wired
up (see `@yokailabs/prettier-config`), and vitest as the test runner. Add
repo-specific jobs alongside the reusable call as needed.

### Pulumi preview

```yaml
name: Pulumi Preview
on:
  pull_request:
    paths: ["infra/pulumi/**"]

jobs:
  preview:
    uses: Yokai-Lab/github-workflows/.github/workflows/pulumi-preview-node.yml@v1
    with:
      stack-name: beta
      work-dir: infra/pulumi
      pulumi-backend-url: s3://my-pulumi-state-bucket
    secrets:
      PULUMI_CONFIG_PASSPHRASE: ${{ secrets.PULUMI_CONFIG_PASSPHRASE }}
```

Set repo/environment variables `AWS_DEPLOY_ROLE_ARN` (an IAM role whose trust
policy allows this repo's GitHub OIDC subject) and, if not passed inline,
`PULUMI_BACKEND_URL`. `aws-region` defaults to `us-east-2`.

### npm release (Changesets + OIDC Trusted Publishing)

Publish-only, **no PRs**, no token — nothing to rotate or expire. Provenance
attestations are automatic. You version locally and commit to main; CI's only
job is the publish.

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]

# Both are required. id-token:write must be on the CALLER too — OIDC won't work
# if only the reusable workflow declares it. contents:write pushes release tags.
permissions:
  contents: write
  id-token: write

jobs:
  release:
    uses: Yokai-Lab/github-workflows/.github/workflows/npm-publish.yml@v1
    # Release tags are committed as "Yokai Labs CI <ci@yokailabs.com>" by
    # default. Override per consumer:
    # with:
    #   git-user-name: Yokai Labs CI
    #   git-user-email: patrick+ci@yokailabs.com
```

**One-time setup per consuming repo:**

1. For each package, configure a Trusted Publisher at npmjs.com
   (**Package → Settings → Trusted Publisher → GitHub Actions**):
   - **Organization / repository** = the _consuming_ repo (e.g. `Yokai-Lab/foo`)
   - **Workflow filename** = `release.yml` — the **caller's** filename, **not**
     `npm-publish.yml`. npm validates the entry-point workflow, not the
     reusable one it calls.
   - **Environment** = leave blank
   - Recommended package setting: **"Require two-factor authentication and
     disallow tokens"** — OIDC is exempt, so CI keeps working while no token can
     ever publish.
2. **Bootstrap** a brand-new package name with one manual `npm publish` first —
   trusted publishing can't create a name that doesn't exist yet. After that,
   every release publishes through CI with no token.

Day-to-day, straight on main:

```sh
npx changeset          # record each change
npx changeset version  # bump version + CHANGELOG, consume the changeset
git commit -am release && git push   # push to main → CI publishes via OIDC
```

### Slack failure notification

```yaml
jobs:
  notify-failure:
    needs: [ci, deploy]
    if: ${{ always() && contains(needs.*.result, 'failure') }}
    uses: Yokai-Lab/github-workflows/.github/workflows/notify-slack-failure.yml@v1
    with:
      service-name: my-app
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Adding a workflow

1. Create it in `.github/workflows/` with a `workflow_call` trigger.
2. Parameterize repo-specific values via `inputs` / `secrets`.
3. Document it here.
4. Cut a release: move the `v1` tag (or bump the major on a breaking change).

## Cutting a release

This repo is versioned by git tag; consumers pin to the major tag.

```sh
# non-breaking change → move the major tag forward
git tag -f v1 && git push origin v1 --force

# breaking change → introduce the next major
git tag v2 && git push origin v2
```

(Optionally also push an immutable `v1.x.y` tag per release for an audit trail.)
