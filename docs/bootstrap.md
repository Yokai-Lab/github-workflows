# Bootstrap — consuming these workflows

One-time setup a project must do before the reusable workflows in this repo
will run. **CI (`ci-node`) needs none of this** — it works on a fresh repo.
The rest depends on which workflows you adopt:

| You're adopting…       | Do steps      |
| ---------------------- | ------------- |
| `ci-node`              | none          |
| `npm-publish`          | 5             |
| `pulumi-preview-node`  | 1, 2, 3, 4    |
| `notify-slack-failure` | 6             |

Replace every `<PLACEHOLDER>`. Region defaults to `us-east-2`. Each AWS step
starts with an identity check — **read the expected output before running it.**

## Prerequisites

- AWS CLI v2, targeting the project's account via a named profile
  (`--profile <AWS_PROFILE>`)
- GitHub CLI (`gh`), authenticated
- Pulumi CLI — `brew install pulumi/tap/pulumi` (for the Pulumi steps)

---

## Step 1 — S3 state bucket for Pulumi

One bucket per account; all stacks store state here (namespaced by project).

```bash
aws sts get-caller-identity --profile <AWS_PROFILE>   # verify: target account

BUCKET="pulumi-state-<PROJECT>"

aws s3api create-bucket --bucket "$BUCKET" --region us-east-2 \
  --create-bucket-configuration LocationConstraint=us-east-2 --profile <AWS_PROFILE>

aws s3api put-bucket-versioning --bucket "$BUCKET" \
  --versioning-configuration Status=Enabled --profile <AWS_PROFILE>

aws s3api put-bucket-encryption --bucket "$BUCKET" --profile <AWS_PROFILE> \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

aws s3api put-public-access-block --bucket "$BUCKET" --profile <AWS_PROFILE> \
  --public-access-block-configuration \
  '{"BlockPublicAcls":true,"IgnorePublicAcls":true,"BlockPublicPolicy":true,"RestrictPublicBuckets":true}'

aws s3api head-bucket --bucket "$BUCKET" --profile <AWS_PROFILE> && echo "OK: $BUCKET"
```

---

## Step 2 — GitHub Actions OIDC provider

Once per account. Every repo deploying to this account shares it. **This is the
prerequisite the `pulumi-preview-node` workflow can't create for you.**

```bash
# Skip creation if this returns a non-empty ARN:
aws iam list-open-id-connect-providers --profile <AWS_PROFILE> \
  --query "OpenIDConnectProviderList[?ends_with(Arn,'token.actions.githubusercontent.com')]"

aws iam create-open-id-connect-provider \
  --url "https://token.actions.githubusercontent.com" \
  --client-id-list "sts.amazonaws.com" \
  --thumbprint-list "d89e3bd43d5d909b47a18977aa9d5ce36cee184c" \
  --profile <AWS_PROFILE>
```

> AWS doesn't actually validate the thumbprint for GitHub (it uses a trusted CA
> library), but the field is required.

---

## Step 3 — IAM deploy role for GitHub Actions

The role `pulumi-preview-node` assumes via OIDC. Trust it to the consuming repo.

```bash
ACCOUNT_ID="<ACCOUNT_ID>"

cat > /tmp/trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike": { "token.actions.githubusercontent.com:sub": ["repo:<ORG>/<REPO>:*"] }
    }
  }]
}
EOF

aws iam create-role --role-name github-actions-deploy \
  --assume-role-policy-document file:///tmp/trust-policy.json --profile <AWS_PROFILE>

# Start broad; tighten to a scoped policy after the first successful deploy.
aws iam attach-role-policy --role-name github-actions-deploy \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess --profile <AWS_PROFILE>

rm /tmp/trust-policy.json

aws iam get-role --role-name github-actions-deploy \
  --query 'Role.Arn' --output text --profile <AWS_PROFILE>   # ← note this ARN for Step 4
```

> **TODO once stable:** replace `AdministratorAccess` with a least-privilege
> policy once you know what Pulumi actually touches.

---

## Step 4 — Pulumi passphrase + GitHub Environment per stack

The preview job runs in `environment: <STACK>` (e.g. `beta`, `prod`), so create
a GitHub Environment per stack and scope its vars/secrets there.

```bash
STACK="<STACK>"                                  # e.g. beta
ROLE_ARN="<role ARN from Step 3>"
BACKEND_URL="s3://pulumi-state-<PROJECT>"
PASSPHRASE="$(openssl rand -base64 32)"          # generate once per stack; keep it safe

gh api --method PUT "repos/<ORG>/<REPO>/environments/${STACK}"

gh variable set AWS_DEPLOY_ROLE_ARN --repo <ORG>/<REPO> --env "$STACK" --body "$ROLE_ARN"
gh variable set PULUMI_BACKEND_URL  --repo <ORG>/<REPO> --env "$STACK" --body "$BACKEND_URL"
gh secret   set PULUMI_CONFIG_PASSPHRASE --repo <ORG>/<REPO> --env "$STACK" --body "$PASSPHRASE"
```

> Use the **same passphrase** across stacks/repos that need to decrypt each
> other's StackReference outputs. Store it somewhere durable — losing it means
> losing access to encrypted Pulumi config.

Then initialise the stack locally (one-time), e.g.:

```bash
PULUMI_BACKEND_URL="$BACKEND_URL" PULUMI_CONFIG_PASSPHRASE="$PASSPHRASE" \
AWS_PROFILE=<AWS_PROFILE> pulumi stack init "$STACK"
pulumi config set aws:region us-east-2
```

---

## Step 5 — npm Trusted Publisher (per package)

So `npm-publish` can publish over OIDC with no token. **Configure this on the
package at npmjs.com**, and mind the caller-filename gotcha documented in the
[README release section](../README.md#npm-release-changesets--oidc-trusted-publishing):

- **Organization / repository** = the consuming repo (`<ORG>/<REPO>`)
- **Workflow filename** = your caller workflow, e.g. `release.yml` — **not**
  `npm-publish.yml`. npm validates the entry-point workflow, not the reusable one.
- **Environment** = blank
- Recommended package setting: **"Require two-factor authentication and disallow
  tokens"** — OIDC trusted publishing is exempt, so CI keeps publishing while no
  token can ever publish and your manual publishes still prompt for 2FA.

Bootstrap a brand-new package name with one manual `npm publish` first —
trusted publishing can't create a name that doesn't exist yet.

> The flow is **publish-only, no PRs**: you version locally
> (`npx changeset version`) and commit to main; CI just runs `changeset publish`.
> No repo settings (not even "Allow Actions to create PRs") are needed — the
> caller workflow's own `permissions:` block grants `contents: write` for tags
> and `id-token: write` for OIDC.

---

## Step 6 — Slack webhook secret

For `notify-slack-failure`. Create an [incoming webhook](https://api.slack.com/messaging/webhooks)
and store it (repo-level, or env-scoped to match the deploy):

```bash
gh secret set SLACK_WEBHOOK_URL --repo <ORG>/<REPO> --body "<WEBHOOK_URL>"
```

---

## Checklist

- [ ] S3 Pulumi state bucket created, versioned + encrypted (Step 1)
- [ ] OIDC provider exists in the account (Step 2)
- [ ] `github-actions-deploy` role created, trust scoped to the repo (Step 3)
- [ ] Pulumi passphrase generated; `AWS_DEPLOY_ROLE_ARN` + `PULUMI_BACKEND_URL`
      vars and `PULUMI_CONFIG_PASSPHRASE` secret set per GitHub Environment (Step 4)
- [ ] npm Trusted Publisher configured per package; new names bootstrapped (Step 5)
- [ ] `SLACK_WEBHOOK_URL` set (Step 6) — if using `notify-slack-failure`

## Notes

- **One state bucket + one deploy role per account**, shared across that
  account's repos. Namespace Pulumi state by project name.
- **Environments = stack names.** `AWS_DEPLOY_ROLE_ARN`, `PULUMI_BACKEND_URL`,
  and `PULUMI_CONFIG_PASSPHRASE` are read from the GitHub Environment matching
  the stack — repo-level values won't be seen by the preview job.
- **No static AWS keys, ever.** Auth is OIDC end to end (AWS role + npm trusted
  publishing). If a workflow asks for an access key, something is misconfigured.
