# Terraform Notes — 04 Variables and Outputs

## Names in a block
```hcl
resource "aws_instance" "web" { ... }
#         └── type ──┘  └ local name ┘
```
- **Type** (`aws_instance`) = what I create. Defined by the provider, I can't invent it.
- **Local name** (`web`) = a name I choose, only used inside Terraform. AWS never sees it. It tells apart two resources of the same type.
- **Address** = type + local name → `aws_instance.web`
- **Attribute** = a value of that resource → `aws_instance.web.id`, `aws_instance.web.private_ip`

| Block | Example | Address |
|---|---|---|
| resource | `resource "aws_instance" "web"` | `aws_instance.web` |
| data | `data "aws_vpc" "default_vpc"` | `data.aws_vpc.default_vpc` |
| module | `module "network" { source = "..." }` | `module.network` |

### `alias` (only for providers!)
Used to have **two configurations of the same provider**, for example two regions:
```hcl
provider "aws" {
  region = "sa-east-1"          # default one
}

provider "aws" {
  alias  = "virginia"
  region = "us-east-1"
}

resource "aws_instance" "x" {
  provider = aws.virginia       # use the second one
}
```

## Variables vs locals vs outputs
```
 inputs                    project                  outputs
 variables.tf  ──►   main.tf (+ locals)    ──►   outputs.tf
 .tfvars / -var
```
| | Variable | Local | Output |
|---|---|---|---|
| What | **Input** from outside | **Internal** value | Value shown **after** apply |
| Can change from outside? | ✅ (tfvars, `-var`) | ❌ | — |
| Declare | `variable "x" { }` | `locals { x = ... }` | `output "x" { value = ... }` |
| Use | `var.x` | `local.x` (singular!) | `terraform output` |

- **Locals** can be calculated: `local.name = "${var.env}-web"`. They are visible in **all the `.tf` files of the folder**.
- **Outputs** show things I only know after creation (IP, DNS name, IDs). They are saved in the state.

## `.tfvars` files = the answers to my variables
- `variables.tf` = the **questions** (name, type, description, default).
- `.tfvars` = the **answers** (`instance_name = "hello-world"`).
- Azure equivalent: ARM `parameters.json` / Bicep `.bicepparam`.

| File | Loaded? |
|---|---|
| `terraform.tfvars` | ✅ Automatically |
| `*.auto.tfvars` | ✅ Automatically |
| any other name (`dev.tfvars`) | ❌ Only with `terraform apply -var-file="dev.tfvars"` |

Typical use: one file per environment → `dev.tfvars`, `prod.tfvars`, same `main.tf`.

## Who wins (weakest → strongest, the last one wins)
```
1. default in variables.tf
2. environment variable TF_VAR_name
3. terraform.tfvars
4. *.auto.tfvars
5. -var-file=...  and  -var="name=value"     ← strongest
```

## Variable with no default (= required)
| Where | What happens |
|---|---|
| My terminal | Terraform **asks me** to type the value (shows the `description`) |
| CI/CD or `-input=false` | ❌ **Fails**: `No value for required variable` |

## Secrets (passwords)
- ❌ **Never** in a `default` or in a `.tfvars` file that goes to git.
- ✅ Use an environment variable: `export TF_VAR_db_pass="..."`
- ✅ Or a secrets manager (AWS Secrets Manager / Azure Key Vault).
- `.env` files only if they are in `.gitignore`.
- `sensitive = true` only **hides** the value in the terminal output.
- ⚠️ Secrets **still end up in the state file** → that's why the state must be encrypted.

## `required_providers` vs `provider`
| | `required_providers` | `provider "aws"` |
|---|---|---|
| Question | **Which** provider and version to download | **How** to configure it |
| Contains | `source`, `version` | `region`, credentials, `alias` |
| Used by | `terraform init` | `plan` / `apply` |
| Where | Inside `terraform { }` | Top level of the file |

Analogy: `required_providers` = install the app, `provider` = the app's settings.

## Order of blocks doesn't matter
- Terraform is **declarative**: it reads **all the `.tf` files in the folder**, builds a **dependency graph** from the references, and creates things in the right order (in parallel when possible).
- I can use `local.x` at the top and declare `locals { }` at the bottom ✅
- File names (`main.tf`, `variables.tf`, `outputs.tf`) are only a convention.
- What does matter:
  - `local.x` (singular), not `locals.x`
  - No cycles (A uses B and B uses A → `Error: Cycle`)
  - Order inside a list: `["a", "b"]` ≠ `["b", "a"]`
- See the graph: `terraform graph`

## Things I confused
- `aws_instance` is the **type**, not the name.
- `alias` is **only for providers**, not for resources or modules (that's the local name).
- Locals are **not** only for one file → they work in all files of the folder.
- The flag is `-var-file=`, not `-var-tf=`.
