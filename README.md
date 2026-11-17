# template-repo-terraform-root

A GitHub repository template for an Azure Terraform root configuration (it
starts as a module plus an example that consumes it), providing pinned tools
(mise), Terraform pre-commit hooks (fmt, validate, tflint w/ the `azurerm` ruleset,
terraform-docs, checkov), CI workflows, and Renovate dependency updates. The
module scaffold creates an `azurerm_resource_group`, named via
[Azure/naming/azurerm](https://registry.terraform.io/modules/Azure/naming/azurerm/latest)
— replace/extend it with real resources as the module grows.

## Getting started

Once per machine, set up the tools with
[jay-withers/toolchain](https://github.com/jay-withers/toolchain)
(`bash src/setup.sh`), which installs [mise](https://mise.jdx.dev). Then, in
this repo:

```bash
mise trust && make install   # make install runs `mise install` first
```

`mise.toml` pins every tool this repo's hooks, build and CI use; CI installs
the same versions (jay-withers/workflows v1.26.1+). Add a tool there rather
than installing it by hand.

## Commands

Run `make` (or `make help`) to list the available targets:

```bash
make install           # install mise tools and pre-commit hooks (run once after cloning)
make lint              # run all pre-commit hooks against every file
make init              # terraform init in terraform/
make fmt               # terraform fmt -recursive
make validate          # terraform init + validate
make plan              # terraform init + plan the basic example (set ENV=dev|stg|prd, default dev)
make test              # terraform test (mocked azurerm provider — no Azure auth needed)
```

`fmt`/`validate`/`test` run against the `terraform/` module directory via
`-chdir`. `plan` runs against `terraform/examples/basic/` instead — the module
itself has no provider block (by design; modules shouldn't configure
providers), so it can't be planned on its own. See [Azure auth for `terraform
plan`](#azure-auth-for-terraform-plan) below: `make plan` creates a real
`azurerm_resource_group`, so — unlike `fmt`/`validate`/`test` — it needs real
Azure credentials. The Terraform version is pinned in `mise.toml`
(used locally and by CI).

## Testing

`terraform/tests/*.tftest.hcl` holds the module's tests. They mock the `azurerm`
provider, so `make test` (and `ci-terraform`) run with no Azure credentials.
Add `assert` blocks as the module grows.

## Environments

`terraform/environments/{dev,stg,prd}.tfvars` hold per-environment inputs for
the module and for root configs like `terraform/examples/basic/`. `make plan`
picks one via `ENV` (default `dev`):

```bash
make plan            # plans the basic example with environments/dev.tfvars
make plan ENV=stg    # plans the basic example with environments/stg.tfvars
make plan ENV=prd    # plans the basic example with environments/prd.tfvars
```

Planning a different root config (e.g. the example) directly needs a
`-var-file` path relative to *that* config's own directory:

```bash
terraform -chdir=terraform/examples/basic plan -var-file=../../environments/dev.tfvars
```

These files are **committed, not gitignored** — see `.gitignore`. Don't put
secrets in them; use Azure Key Vault, GitHub Actions secrets, or `ARM_*`
environment variables instead. `gitleaks` (pre-commit and CI) scans every
commit as a backstop. The module's `environment` input (`terraform/variables.tf`)
validates against exactly these three values.

## File layout convention

Every root config and the module itself follow a fixed layout, enforced (not
just documented): `locals`/`variable`/`output`/`data` blocks must live in a
matching `locals.tf`/`variables.tf`/`outputs.tf`/`data.tf`, and
`terraform{}`/`provider{}` blocks in `versions.tf` — or a topic-scoped variant
of any of them (e.g. `outputs.network.tf`, `data.state.tf`) — via the local
pre-commit hook `check-tf-standards` (`scripts/check-tf-standards.sh`).
`main.tf` is left for resources and module blocks.

## Linting against every environment

`make lint` runs TFLint and Checkov once per file in `terraform/environments/`
(via `scripts/tflint-per-env.sh` and `scripts/checkov-per-env.sh`, wired in as
local pre-commit hooks) rather than once with unresolved variables. Passing
each environment's real `-var-file` gives rules that depend on concrete
values — resource naming, tags, region-specific checks — something to
actually evaluate. Both scripts glob `terraform/environments/*.tfvars` at
run time, so adding a new environment's tfvars file is enough to get it
linted — no `.pre-commit-config.yaml` change needed. `tflint-per-env.sh` also
lints every directory under `terraform/` with `.tf` files (the module and
`terraform/examples/basic/`, and any future example) since tflint doesn't
recurse into subdirectories the way Checkov does.

Checkov does *not* run with `--download-external-modules` — it was tried, but
cost ~15s per invocation regardless of caching (checkov's own graph-building
overhead, not network time) for zero benefit, since `Azure/naming/azurerm` has
no resources of its own to find. Checkov logs a harmless "Failed to download
module" warning as a result; revisit if the module tree grows to include
modules with real resources.

## Tags and resource protection

The module applies default tags (`environment`, `managed-by = "terraform"`) to
every resource it creates, merged with — and overridable by — `var.tags`. See
`terraform/locals.tf`'s `local.default_tags`.

Azure-side deletion protection (`prevent_deletion_if_contains_resources`) is a
provider `features` block setting, so it can't live in the module itself
(modules shouldn't configure providers). `terraform/examples/basic/main.tf`
sets it explicitly — it already defaults to `true` in the `azurerm` provider,
but consumers of this module should set it in their own root provider block
too, since the module can't do it for them.

## Azure auth for `terraform plan`

`make plan` creates a real `azurerm_resource_group`, so it needs real Azure
credentials: whatever the `azurerm` provider picks up normally, `az login` or
`ARM_*` environment variables. `terraform test` (mocked provider) is the only
credential-free check, and the only one CI runs on a fresh repo.

## CI/CD

Every workflow is a thin caller of a shared one in
[jay-withers/workflows](https://github.com/jay-withers/workflows), pinned by
SHA. A repo made from this template is green from its first PR, with no Azure
auth configured.

- **ci-pre-commit**: every hook, Terraform ones included. Check:
  `pre-commit / Pre-commit`.
- **ci-terraform**: credential-free `init` + `validate` of the module and the
  example, and `terraform test` of the module. Check: `terraform / Terraform`
  (an always-reporting gate, so it can be required). There is no plan job.
- **ci-codeql**: CodeQL code scanning of the workflows.
- **cd-tag**: a semver tag and release on every merge to `main`.
- **cd-terraform**, **schedule-drift** and **manual-terraform-unlock** are
  dormant until `terraform/backends/dev.hcl` exists. Then cd-terraform plans
  dev on merge, opens a `terraform-pending` issue when the plan has changes,
  and applies when run by hand from the Actions tab; schedule-drift plans
  nightly and records drift; manual-terraform-unlock releases a stuck state
  lock. Only dev is ever applied.

To make a repo created from this template deploy for real:

1. In [github-repos](https://github.com/jay-withers/github-repos), add the
   repo to `state_consumers` with its `deploy_resource_groups` and
   `create_resource_groups = true`, and apply. That creates the resource
   group, the OIDC identity and its grants, and sets `AZURE_CLIENT_ID`,
   `AZURE_TENANT_ID` and `AZURE_SUBSCRIPTION_ID` on the repo's `dev`
   environment.
2. Make `terraform/` the root module (provider block, `backend "azurerm" {}`)
   and add `terraform/backends/dev.hcl`.
3. Merge, then run cd-terraform to apply.

The identity holds Contributor and conditioned RBAC Administrator on its own
resource group(s) and Reader on the subscription, so role assignments outside
its group still need a local `terraform apply`. Grant roles to explicit object IDs,
never to "whoever applies".

## Configuring GitHub for a repo created from this template

Branch protection and other repo-level GitHub settings aren't templated as
files here. This template's own repo is managed centrally by
[jay-withers/github-repos](https://github.com/jay-withers/github-repos)'s
Terraform root module, which is the single source of truth for every
jay-withers repo. A brand-new repo created from this template starts out with
GitHub's defaults and picks up the same managed settings once it's added to
that module's `var.repos` and applied.

## Structure

```text
mise.toml              # pinned tool versions: terraform, tflint, terraform-docs, checkov, pre-commit
terraform/
  versions.tf          # required_version / required_providers
  main.tf              # resources (azurerm_resource_group, named via Azure/naming/azurerm)
  locals.tf            # local values (e.g. default tags)
  variables.tf         # inputs (environment, location, tags)
  outputs.tf           # outputs
  README.md            # generated by terraform-docs
  .terraform-docs.yml  # terraform-docs config (also in examples/basic/)
  .tflint.hcl          # TFLint config (terraform + azurerm rulesets)
  tests/
    defaults.tftest.hcl  # terraform test (mocked azurerm provider)
  environments/
    dev.tfvars         # -var-file inputs per environment (committed, not gitignored)
    stg.tfvars
    prd.tfvars
  examples/
    basic/             # example `make plan` plans against (main.tf, variables.tf, outputs.tf)
.pre-commit-config.yaml
commitlint.config.js
.vscode/
  extensions.json      # recommended VS Code extensions
.github/
  workflows/
    ci-pre-commit.yml  # lints all files on PRs to main
    ci-terraform.yml   # validate module + example, terraform test (no credentials)
    ci-codeql.yml      # CodeQL over the workflows
    cd-tag.yml         # auto-tags on merge to main (semver patch bump)
    cd-terraform.yml   # plans dev on merge, applies on manual run; dormant until terraform/backends/dev.hcl exists
    schedule-drift.yml # nightly drift plan; dormant likewise
    manual-terraform-unlock.yml  # releases a stuck state lock; dormant likewise
renovate.json          # automated dependency updates
scripts/
  check-tf-standards.sh     # pre-commit hook: enforces the Terraform file layout standards
  tflint-per-env.sh         # pre-commit hook: tflint once per terraform/environments/*.tfvars
  checkov-per-env.sh        # pre-commit hook: checkov once per terraform/environments/*.tfvars
Makefile
```
