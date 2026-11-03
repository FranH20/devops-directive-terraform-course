# Terraform Notes — 05 Language Features

## `count` vs `for_each`
Both create several copies of a resource. The difference is how Terraform identifies each copy.

| | `count` | `for_each` |
|---|---|---|
| Identifies copies by | **Index** (position) | **Name** (key) |
| Reference | `aws_instance.web[0]` | `aws_instance.web["api"]` |
| Current item | `count.index` | `each.key` / `each.value` |
| Best for | On/off, identical copies | Copies with their own name or settings |

```hcl
resource "aws_instance" "with_count" {
  count         = length(var.names)
  ami           = var.ami
  instance_type = "t3.micro"
  tags          = { Name = var.names[count.index] }
}

resource "aws_instance" "with_for_each" {
  for_each      = toset(var.names)
  ami           = var.ami
  instance_type = "t3.micro"
  tags          = { Name = each.key }
}
```

### Removing "api" from `["api", "worker", "db"]`
**With `count`, the copies after the removed item shift:**
```
web[0]   api    → worker   ~ changes
web[1]   worker → db       ~ changes
web[2]   db     → (none)   - destroyed
```
- Shifted copies are updated in place or recreated, depending on the setting that changed.
- Copies **before** the removed item don't change. All three change here only because "api" is first.

**With `for_each`, only the one I removed changes:**
```
web["api"]      - destroyed
web["worker"]   no change
web["db"]       no change
```

**Rule:** if each copy has its own name or settings → `for_each`.

## Optional resource
```hcl
resource "aws_db_instance" "db" {
  count = var.create_db ? 1 : 0     # 1 = create it, 0 = don't create it
  # ...
}
```
`var.create_db == true` also works, but it isn't needed because the variable is already true or false.

## Expressions
```hcl
# Conditional: condition ? value_if_true : value_if_false
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"   # text needs quotes!

# String template
name = "${var.env}-web-server"

# Splat: one attribute from all the copies (returns a list)
aws_instance.web[*].id                   # with count
values(aws_instance.web)[*].id           # with for_each
```

## `lifecycle`
Controls how Terraform creates, updates and destroys a resource.

| Option | What it does |
|---|---|
| `prevent_destroy` | Terraform refuses any plan that would destroy this resource |
| `create_before_destroy` | Creates the replacement first, then destroys the old one |
| `ignore_changes` | Terraform ignores changes to the listed settings and doesn't revert them |

- `ignore_changes` is **not a lock**: the resource can still change. Terraform just doesn't react.
- `prevent_destroy` only protects against Terraform. Someone can still delete the resource in the AWS console.

```hcl
lifecycle {
  create_before_destroy = true
  ignore_changes        = [tags]
}
```

## `depends_on`
Only for **hidden dependencies**: when one resource needs another, but doesn't reference it.

```hcl
# A reference already creates the dependency → no depends_on needed
subnet_id = aws_subnet.main.id

# Hidden dependency: the app needs this policy, but nothing references it
resource "aws_instance" "app" {
  # ...
  depends_on = [aws_iam_role_policy.app_policy]
}
```

## Test without creating anything
```bash
terraform console
> upper("hello")
"HELLO"
```

## Things I confused
- The current item is `each.key` / `count.index`, not `splat()` / `length()`.
- `count = var.create_db ? 1 : 0` uses **numbers**, not create / notcreate.
- Text values in a conditional need **quotes**.
- `ignore_changes` doesn't lock the resource.
- `depends_on` is **not** for references. References already create the dependency. It's for hidden dependencies.

## To practice
- [ ] Write a `for_each` and a `count` resource from memory
- [ ] Use `each.key` and `count.index`
- [ ] Find a real example of a hidden dependency for `depends_on`
