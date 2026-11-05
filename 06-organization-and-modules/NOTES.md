# Terraform Notes — 06 Organization and Modules

## What is a module?
A **group of resources** that I can **reuse and customize with variables**. One definition, many copies.

- **Root module** = the folder where I run `terraform`.
- **Reusable (child) module** = the module that a root module calls.
- A module uses **all the `.tf` files in its folder** (`compute.tf`, `database.tf`, `dns.tf`...).

In this lesson there are **two root modules**:
| Root module | Calls | Type |
|---|---|---|
| `consul/` | `terraform-aws-consul` from GitHub | External |
| `web-app/` | `../web-app-module` | Local |

## Calling a module
```hcl
module "web_app_1" {
  source = "../web-app-module"      # where the module is

  app_name      = "web-app-1"       # inputs
  instance_type = "t2.micro"
}
```

### `source`
- **Local** paths must start with `./` or `../`.
- `source = "web-app-module"` ❌ → Terraform looks for it in the online registry.
- `source = "../web-app-module"` ✅

### When to run `terraform init` again
| Change | `init` needed? |
|---|---|
| Add a module | ✅ |
| Change a module's `source` or version | ✅ |
| Edit the `.tf` files inside a local module | ❌ |

## Inputs and outputs
```
web-app/main.tf                  web-app-module/
(root module)                    (reusable module)

module "web_app_1"  ── inputs ─► variables.tf
                                 compute.tf, database.tf, dns.tf...
module.web_app_1.x  ◄─ outputs ─ outputs.tf
```

### Inputs
- I **pass** the value in the `module` block in the root module.
- The variable must be **declared** in the module's own `variables.tf`.

### Required variables (no default)
| Variable is in | Value missing → |
|---|---|
| **Root** module | Terraform asks me in the terminal |
| **Reusable** module | ❌ Error: `Missing required argument` |

The root module can ask for the value and pass it along:
```hcl
variable "db_pass_1" {
  type      = string
  sensitive = true
}

module "web_app_1" {
  source  = "../web-app-module"
  db_pass = var.db_pass_1
}
```

### Outputs
The root module can only read what the module exposes:
```hcl
# web-app-module/outputs.tf
output "instance_1_ip_addr" {
  value = aws_instance.instance_1.public_ip
}

# web-app/main.tf
module.web_app_1.instance_1_ip_addr    # ✅ exposed by the module
aws_instance.instance_1.public_ip      # ❌ not visible from web-app/
```

To print it after `apply`, the root module needs its own output:
```hcl
output "app_1_ip" {
  value = module.web_app_1.instance_1_ip_addr
}
```

## Calling the same module twice
Each call creates its own copy of all the resources, using different input values:
```hcl
module "web_app_1" {
  source   = "../web-app-module"
  app_name = "web-app-1"
}

module "web_app_2" {
  source   = "../web-app-module"
  app_name = "web-app-2"
}
```
Here they are two apps in `production`. The same pattern can create different environments, such as dev and prod.

## What goes where
| Block | Where | Why |
|---|---|---|
| `backend` | Root module | The root module decides where the state is stored |
| `provider "aws"` (region, credentials) | Root module | The root module decides where to deploy |
| `required_providers` | Root and reusable module | Each one declares which providers it needs |
| Resources | Reusable module | They are the reusable part |

That's why the README says to remove the backend and provider definitions from the module. This way it can be reused anywhere.

## Optional resource inside a module
```hcl
resource "aws_route53_zone" "primary" {
  count = var.create_dns_zone ? 1 : 0   # true = create it, false = skip it
  name  = var.domain
}

aws_route53_zone.primary[0].zone_id
```
A resource with `count` is a **list**, even with one copy, so I need `[0]`.

## Things I confused
- Inputs are **passed** in the `module` block, but **declared** in the module's `variables.tf`.
- A reusable module's missing variable **doesn't prompt**. It gives an error.
- To read a value from a module: `module.<name>.<output>`.

## To review later
- [ ] When `terraform init` is needed for a module
- [ ] Why `backend` and `provider` belong in the root module
- [ ] `count` and `[0]` in `web-app-module/dns.tf`
