# Terraform Notes — 03 Basics (Remote Backend)

## Why store the state in S3 (remote backend)
- **Teamwork:** other people only need to connect to the bucket to get the same state.
- **Locking:** if someone is running `apply`, the state is locked and another person's `apply` has to wait. This avoids corrupting the state.
- **Safe place:** the state is in AWS, encrypted and versioned, not only on my laptop (if my laptop breaks, the state is not lost).

## Locking: DynamoDB vs `use_lockfile`
- **Old way:** a **DynamoDB** table was used for the **lock** (not for encryption).
- **New way (Terraform 1.10+):** `use_lockfile = true` → the lock is a small `.tflock` file in the same S3 bucket. No DynamoDB needed.
- **Encryption** is a different thing: `encrypt = true` in the backend + encryption config on the bucket.

```
 terraform apply
      ├─1─► lock the state (.tflock)    (someone else has it → wait / error)
      ├─2─► download terraform.tfstate from S3
      ├─3─► provider makes the changes in AWS
      ├─4─► upload the new terraform.tfstate to S3
      └─5─► unlock
```

## Bootstrap: why 2 steps (chicken and egg)
The backend needs the bucket to **already exist**, but I create the bucket with Terraform.
1. **Step 1:** backend commented out → `init` + `apply` → creates the S3 bucket (state is still local).
2. **Step 2:** uncomment the `backend "s3"` block → `terraform init` → answer `yes` → my state is copied into the bucket I just created.

`use_lockfile` removes the DynamoDB table, **not** the need to create the bucket first.

## The backend block
```hcl
terraform {
  backend "s3" {
    bucket       = "franklin-tf-state-8k3x"                       # which bucket
    key          = "03-basics/import-bootstrap/terraform.tfstate" # path inside the bucket
    region       = "sa-east-1"
    use_lockfile = true
    encrypt      = true
  }
}
```
- **`key`** = the path where my state file will be saved **inside the bucket**.
  - It does **not** need to match my local folder name.
  - The file doesn't need to exist before: Terraform creates it the first time it saves the state.
  - S3 has no real folders, the `/` is just part of the name.
  - **Each project needs a different key**, or they overwrite each other's state.
- Terraform only saves **the state** there. My `main.tf` is not uploaded.

## `backend "s3"` vs `resource "aws_s3_bucket"`
| | `resource "aws_s3_bucket"` | `backend "s3"` |
|---|---|---|
| What it is | Config to **create** the bucket | Setting that says **where to save my state** |
| Where it goes | Top level of the file | Inside the `terraform { }` block |
| When it's read | `plan` / `apply` | `init`, **before anything else** |

- The backend is a **setting of Terraform itself**, not a resource (and not a module — module is another Terraform concept).
- The backend **can't use references or variables** like `aws_s3_bucket.terraform_state.bucket`, because it's read before Terraform knows any resources. Only fixed values → that's why the bucket name is written **twice** and must match.

## When to run `terraform init` again
Only when I change: the **provider** (or its version), the **backend**, or **modules**.
Changing a resource value (like the bucket name) → just `plan` / `apply`.

## Things I learned by failing
- **S3 bucket names are unique in all of AWS** (all accounts in the world) → `devops-directive-tf-state` was already taken. Use my own name: lowercase, numbers, hyphens, 3–63 chars.
- `use_locking` doesn't exist → the correct name is **`use_lockfile`**.
- Provider upgrade 3.x → 6.x broke old code: `aws_subnet_ids` → `aws_subnets`, RDS `name` → `db_name`, `db.t2.micro` → `db.t3.micro`.

## To review later
- [ ] Path of a request in the web-app (Route 53 → ALB → EC2) — see `web-app/architecture.png`
- [ ] `resource` vs `data` block
- [ ] Why the browser timed out but `curl http://` worked (browser forces HTTPS, only port 80 was open)
- [ ] Correct order to destroy a bootstrap backend (destroy `web-app` first; move state back to local before destroying the bucket)
