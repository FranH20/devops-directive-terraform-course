# Terraform Notes — 02 Overview

## The 4 main pieces

- **Config** (`main.tf`) = the configuration I *want* for my resources. I describe *what*, not *how*.
- **State** (`terraform.tfstate`) = Terraform's memory of the resources it already created (IDs, IPs, settings). Terraform doesn't remember on its own what exists, so it saves it here.
- **Terraform Core** = the `terraform` program. It compares config vs state and makes the plan (create / update / delete).
- **Provider** = plugin that knows how to talk to one platform's API (AWS, Azure, GCP...). Core plans, the provider makes the real API calls.

```
 Config (.tf)  ──┐
                 ├──►  Terraform Core  ──►  Provider (aws)  ──►  AWS API
 State (.tfstate)┘     "what changed?"      "make API call"      (real servers)
                              ▲                                        │
                              └────────── updates state ◄──────────────┘
```

> Config = what I **want**. State = what **exists**. Core finds the **difference**. Provider **makes the change**.

## How a change works
1. I edit `main.tf` (example: `t2.micro` → `t3.micro`).
2. `terraform plan` → Core compares my config with the state (and checks the real resources through the provider).
3. It shows the difference: `+` create, `~` update, `-` destroy.
4. `terraform apply` → the provider makes the change, and the state is updated.

## Commands
| Command | What it does |
|---|---|
| `terraform init` | Prepares the folder: downloads providers into `.terraform/` and creates `.terraform.lock.hcl`. Safe to run many times. |
| `terraform plan` | Shows what would change. Doesn't change anything. |
| `terraform apply` | Makes the changes for real. |
| `terraform destroy` | Deletes everything this config created. **Run it when I finish practicing!** |

## Files in the folder
| File | What it is | Commit to git? |
|---|---|---|
| `main.tf` | My config | ✅ Yes |
| `.terraform/` | Downloaded providers (big, can be ~500 MB) | ❌ No, `init` recreates it |
| `.terraform.lock.hcl` | Records the **exact provider versions** chosen (+ checksums) so next time / teammates get the same ones | ✅ Yes (normally) |
| `terraform.tfstate` | State. Can contain secrets | ❌ Never. Never edit it by hand |

## The resource block
```hcl
resource "aws_instance" "example" {
  ami           = "ami-06ae896a166ca24f2"  # Ubuntu, sa-east-1
  instance_type = "t2.micro"               # size: 1 vCPU, 1 GB RAM
}
```
- `aws_instance` = the type (`aws_` → comes from the AWS provider; instance = EC2 server).
- `example` = my name for it inside Terraform → address `aws_instance.example`.
- **AMI** = the operating system image. **AMI IDs are different in each region**, so if I change the region I must change the AMI too.

## Find an Ubuntu AMI for a region
```bash
aws ec2 describe-images --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd*/ubuntu-*-24.04-amd64-server-*" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text --region sa-east-1
```
(`099720109477` = Canonical, the company that publishes Ubuntu.)

## Setup (Mac)
- Terraform: `brew tap hashicorp/tap && brew install hashicorp/tap/terraform`
- AWS CLI: `brew install awscli`, then `aws configure` (access keys) or `aws login` (browser)
- Check the login: `aws sts get-caller-identity`
