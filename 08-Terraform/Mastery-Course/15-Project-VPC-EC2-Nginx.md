---
title: "15 — Project: VPC + EC2 + NGINX"
type: project
module: terraform
status: complete
interview-relevance: ★★★★☆
tags:
  - terraform
  - project
  - ec2
  - nginx
  - user-data
  - security-groups
config: aws-vpc-ec2-nginx/
up: "[[Terraform-Mastery-MOC]]"
prev: "[[14-Project-S3-Static-Website]]"
next: "[[16-Project-IAM-YAML-Automation]]"
origin: Terraform-Mastery/15-Project-VPC-EC2-Nginx.md
---

# 15 — Project: VPC + EC2 + NGINX

← Back to [[Terraform-Mastery-MOC]]

> **This is the config to walk through when asked "tell me about something you built with Terraform."** It's end-to-end: network, compute, security, bootstrap, output.

---

## Structure — multi-file, split by domain

```
aws-vpc-ec2-nginx/
├── main.tf              # terraform{} block only
├── providers.tf         # provider "aws"
├── vpc.tf               # VPC, subnets, IGW, route tables
├── security_groups.tf   # nginx SG
├── ec2.tf               # instance + user_data
└── outputs.tf           # public IP + URL
```

**The point of this layout:** resources in `ec2.tf` reference resources in `vpc.tf` with no imports, no ordering directives, nothing. Terraform loads every `.tf` in the directory as one configuration. File names are purely for humans.

---

## The config

**`vpc.tf`** — identical to [[13-AWS-Networking-VPC-Concepts]]: VPC `10.0.0.0/16`, private subnet `10.0.1.0/24`, public subnet `10.0.2.0/24` with `map_public_ip_on_launch = true`, IGW, route table with `0.0.0.0/0 → IGW`, association to the public subnet.

**`security_groups.tf`**
```hcl
resource "aws_security_group" "nginx-sg" {
  vpc_id = aws_vpc.my_vpc.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"       # -1 = all protocols
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "nginx-sg" }
}
```

**`ec2.tf`**
```hcl
resource "aws_instance" "nginxserver" {
  ami                         = "ami-0c0e147c706360bd7"
  instance_type               = "t3.nano"
  subnet_id                   = aws_subnet.public-subnet.id
  vpc_security_group_ids      = [aws_security_group.nginx-sg.id]
  associate_public_ip_address = true

  user_data = <<-EOF
            #!/bin/bash
            sudo yum install nginx -y
            sudo systemctl start nginx
            EOF

  tags = { Name = "NginxServer" }
}
```

**`outputs.tf`**
```hcl
output "instance_public_ip" {
  description = "The public IP address of the EC2 instance"
  value       = aws_instance.nginxserver.public_ip
}

output "instance_url" {
  description = "The URL to access the Nginx server"
  value       = "http://${aws_instance.nginxserver.public_ip}"
}
```

---

## `user_data` — the bootstrap mechanism

Runs **once, at first boot**, as root.

**Heredoc syntax:**
- `<<-EOF` — the dash strips leading whitespace so you can indent the block naturally
- `<<EOF` — preserves indentation exactly (breaks scripts that care)
- The shebang `#!/bin/bash` is required

**Where to debug it:**
```bash
/var/log/cloud-init-output.log   # ← the one you actually want; shows script stdout/stderr
/var/log/cloud-init.log          # cloud-init's own operation log
```

> **"nginx isn't running after apply" is a standard scenario question.** The answer that shows experience: Terraform reports success as soon as the API call to create the instance succeeds — it does **not** wait for user_data to finish or check whether it worked. So a failed bootstrap looks like a successful apply. SSH in, read `/var/log/cloud-init-output.log`.

**Changing `user_data` forces instance replacement.** `user_data_replace_on_change` controls this explicitly.

**Better pattern — `templatefile()`:**
```hcl
user_data = templatefile("${path.module}/init.sh.tftpl", {
  app_port = var.app_port
  env      = var.environment
})
```
Keeps the script in its own file (shellcheck-able, syntax-highlighted) and injects Terraform values into it. Far better than an inline heredoc once the script is more than three lines.

**Production reality:** user_data is fine for bootstrapping. For anything substantial you'd use a Packer-built golden AMI with software pre-installed, or a config management tool, or skip instances entirely and use containers. Saying this unprompted is a strong signal.

---

## What's wrong with this config

Being able to critique it is worth more in an interview than the config itself.

| Issue | Fix |
|---|---|
| **`0.0.0.0/0` on port 80** | Acceptable for a public web server, but only because it's port 80. Never do this for 22 or 3389. |
| **No SSH access at all** | No `key_name` set — can't get in to debug the nginx bootstrap. Ironic given the failure mode above. |
| **`yum install nginx` on Amazon Linux 2** | Fails. Needs `amazon-linux-extras install nginx1`. On AL2023 it's `dnf install nginx`. **This is an actual bug in the course config.** |
| **`systemctl start` without `enable`** | nginx won't survive a reboot. Needs `systemctl enable --now nginx`. |
| **Hardcoded AMI** | Region-locked. Use `data "aws_ami"` → [[06-Data-Sources]] |
| **Single instance, single AZ** | No HA. Production = ALB + ASG across ≥2 AZs. |
| **Public IP, no Elastic IP** | IP changes on stop/start, breaking any DNS pinned to it. |
| **No HTTPS** | Port 443 + ACM certificate + ALB. |
| **No monitoring** | No CloudWatch agent, no alarms. |

**The fixed user_data:**
```bash
#!/bin/bash
set -euxo pipefail                        # fail loudly, log every command
dnf install -y nginx || amazon-linux-extras install -y nginx1
systemctl enable --now nginx              # start AND survive reboot
```
`set -euxo pipefail` is the detail that matters — without it a failing command is silently skipped and you get a half-bootstrapped instance that looks fine.

---

## Production evolution

```
Route53
   ↓
ALB (HTTPS via ACM, public subnets across 2+ AZs)
   ↓
Target Group
   ↓
Auto Scaling Group (private subnets across 2+ AZs)
   ↓
EC2 instances from a Launch Template + golden AMI
   ↓
NAT Gateway (per AZ) for outbound
```

Security group chain — the pattern worth knowing:
- ALB SG: inbound 443 from `0.0.0.0/0`
- Instance SG: inbound 80 **from the ALB's security group**, not from a CIDR

**Referencing a security group as a source rather than a CIDR is the good practice.** It means the rule stays correct even as the ALB's IPs change, and it expresses intent — "traffic from the load balancer" rather than "traffic from this IP range".

---

## Real-world mapping

| Concept | Job reality |
|---|---|
| Multi-file split by domain | Standard once a config exceeds one screen |
| `user_data` | Bootstrapping; but golden AMIs or containers for anything substantial |
| Outputting a usable URL | Good habit — makes the apply immediately verifiable |
| SG referencing another SG | The correct tiered-architecture pattern |
| Single EC2 + nginx | Only appropriate for a lab or a bastion. Real workloads are ASG or containers. |

**Honest positioning for interviews:** this project demonstrates the full Terraform loop — network, compute, security, bootstrap, output. It's a lab, not production, and saying so while explaining exactly what production would add is stronger than overselling it.

---

## Interview Q&A

**Q: Walk me through provisioning a web server on AWS with Terraform.**
> I'd build the network first — a VPC, a public subnet, an internet gateway, and a route table with a default route to the gateway associated to that subnet, since the route is what actually makes the subnet public. Then a security group allowing inbound 80, and an EC2 instance in that public subnet referencing the security group, with a user_data script to install and start nginx. Finally an output with the URL so the apply is immediately verifiable. Terraform works out the ordering itself from the references between the resources — I never specify that the VPC comes before the subnet, that falls out of the subnet referencing the VPC's ID.

**Q: You applied successfully but the nginx page doesn't load. How do you debug it?**
> The key thing to know first is that a successful apply doesn't mean the bootstrap worked — Terraform returns as soon as the instance creation API call succeeds, and it never checks whether user_data completed. So I'd work outside in. Is the instance actually running and does it have a public IP? Does the security group allow 80 inbound? Is the subnet's route table pointing at an internet gateway? Then, if all the network layers look right, I'd SSH in and read `/var/log/cloud-init-output.log`, which has the user_data script's stdout and stderr. The failure mode narrows it fast: a timeout means traffic is being dropped, so it's the security group or routing; connection refused means it reached the host and nothing is listening, so nginx didn't start.

**Q: How does user_data work and what are its limits?**
> It's a script passed to the instance at launch and executed once at first boot as root, via cloud-init. The limits are that there's no feedback loop — Terraform doesn't wait for it or know if it failed — there's a 16KB size cap, it only runs on first boot by default, and changing it forces instance replacement. For anything beyond a few lines I'd use `templatefile()` so the script lives in its own file and can be linted, with Terraform values injected into it. For production I'd generally move the software installation into a Packer-built AMI so instances boot ready, which also makes autoscaling faster since there's no install step on launch.

**Q: What's wrong with `cidr_blocks = ["0.0.0.0/0"]` in a security group?**
> It depends entirely on the port. For port 80 or 443 on a public web server it's correct — that's what public means. For SSH on 22 or RDP on 3389 it's a serious finding and any security scanner will flag it; those should be restricted to a known office CIDR or VPN range, or better, replaced with SSM Session Manager so no inbound port is open at all. The broader pattern I'd use in a tiered architecture is referencing another security group as the source rather than a CIDR — so the instance security group allows port 80 from the load balancer's security group. That stays correct as IPs change and it expresses the actual intent.

**Q: Why split the config across vpc.tf, ec2.tf, security_groups.tf instead of one main.tf?**
> Purely for humans — Terraform loads every .tf file in the directory and treats them as one configuration, so the file names carry no meaning to it. Resources in ec2.tf reference resources in vpc.tf with no imports needed. The benefit is navigability and cleaner diffs in code review: a change to networking touches vpc.tf and a reviewer knows immediately what layer is affected. I split by domain rather than by block type, so networking, compute, and security rather than resources.tf and variables.tf, once the config is more than a screen long.

---

## Gotchas & flags

- `yum install nginx` **fails on Amazon Linux 2** — needs `amazon-linux-extras install nginx1`. AL2023 uses `dnf`. The course config has this bug.
- `systemctl start` without `enable` means nginx dies on reboot.
- `<<-EOF` (with dash) strips leading whitespace; `<<EOF` doesn't. Use the dash form for indented heredocs.
- Changing `user_data` **replaces the instance**.
- user_data runs **once**, at first boot only, unless explicitly configured otherwise.
- No `key_name` means no SSH, ever — and you can't add it after launch.
- `set -euxo pipefail` at the top of every bootstrap script. Without it, silent partial failures.

---

Next: [[16-Project-IAM-YAML-Automation]]
