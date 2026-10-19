# mmoa_web_app_infra

AWS CloudFormation templates for the **mmoa web app**, the website for the
[Mission Men of Anna](https://www.facebook.com/missionmenofanna) (MMoA), a
501(c)(3) nonprofit in Anna, Texas:

> Our mission is to come together as men in service of those in need by
> organizing events to provide essential resources to support the community
> and the people of Anna.

Modeled on
`car_show_management_infra`: same file layout, the same naming scheme, and
the same input fields, with `ProjectName` defaulting to **`mmoa`**.

One thing runs here: the **mmoa_web_app**, a single EC2 instance in a private
subnet behind an Application Load Balancer. It uses a local **SQLite**
database, the same engine as `car_show_control_plane`, and is backed up
regularly by the admin console.

Unlike the myevent365 control plane, **nothing in this project can create or
modify AWS resources.** Both IAM roles are limited to SSM plus read-only
calls.

Every stack takes the same `ProjectName` parameter and finds the others
through cross-stack exports and imports. Region: `us-east-1`.

---

## App contract (mmoa_web_app)

These values come from the `mmoa_web_app` repo. If the app changes one of
them, change it here too:

| Setting | Where in infra | Value |
|---------|----------------|-------|
| App listen port (`config.json` → `port`) | `c_security_groups` → `AppPort` (the ALB imports it) | `3004` |
| SQLite setup script | `d_EC2_web_app` → `DbSetupScript` | `SETUP_SQLLITE_DB.sh` |
| App start script | `d_EC2_web_app` → `StartScript` | `START_APP.sh` |
| Public URL baked into verification / reset links | `d_EC2_web_app` → `PublicBaseUrl` (optional; blank keeps `http://localhost:3004`) | blank |
| Health check path | `e_ALB_mmoa` → `HealthCheckPath` | `/health` |

The committed `config.json` already runs correctly behind the ALB (`server.ssl`
false, `server.trustProxy` 1), so user data only edits `publicBaseUrl`. Change
anything else afterwards from the admin console (Web App Settings), then
restart the app from there.

Email is not set up yet, so the committed config has
`email.showLinksInBrowser: true`. With that on, verification and
password-reset links are shown on screen instead of only being emailed.
Turn it off once real email works.

---

## Architecture

```
                 Internet
                    │  HTTP :80   (HTTPS :443 later)
                    ▼
        ┌──────────────────────────┐
        │ mmoa-webapp-alb          │
        │ (public subnets, 2 AZs)  │
        └────────────┬─────────────┘
                     │ HTTP :<AppPort>
                     ▼
        ┌──────────────────────────┐        ┌────────────────────────┐
        │ mmoa_web_app EC2         │◄─SSM───│ mmoa_admin_console     │
        │ t3.small, private subnet │        │ (off-AWS for now)      │
        │ SQLite (local)           │        │ config / logs / backups│
        └────────────┬─────────────┘        └────────────────────────┘
                     │ outbound via NAT Gateway (AZ1)
                     ▼
                  Internet
```

---

## Stack inventory

The letter prefix is the **deploy wave**. Stacks with the same prefix don't
depend on each other.

| File | What it provisions | Depends on |
|------|--------------------|------------|
| [`b_network.yaml`](templates/b_network.yaml) | Small VPC `10.1.0.0/24`, 2 public + 2 private `/26` subnets across `us-east-1a/b`, IGW, single NAT Gateway in AZ1, route tables | none |
| [`b_IAM.yaml`](templates/b_IAM.yaml) | `mmoa_ec2_web_app` role/profile (SSM core only). `mmoa_admin_console` **managed policy** (SSM Run Command to the instance tagged `mmoa_web_app` + read-only EC2/ELB). `mmoa_ec2_adminconsole` role/profile (SSM core + that policy). That role isn't attached to anything yet because the console runs off-AWS; attach the managed policy to the console's IAM user with `aws iam attach-user-policy` | none |
| [`c_security_groups.yaml`](templates/c_security_groups.yaml) | Web app SG `mmoa_ec2_web_app_sg` (self-ingress on `AppPort`) and ALB SG `mmoa_alb_inbound_sg` (80/443 from `0.0.0.0/0`) | `b_network` |
| [`d_EC2_web_app.yaml`](templates/d_EC2_web_app.yaml) | `mmoa_web_app`: single `t3.small` AL2023 instance, `8 GiB gp3` root, private subnet 1, SSM-managed, SQLite | `b_IAM`, `b_network`, `c_security_groups` |
| [`e_ALB_mmoa.yaml`](templates/e_ALB_mmoa.yaml) | `mmoa-webapp-alb` + HTTP:80 listener **forwarding** to `mmoa-webapp-tg`, which registers the instance on `AppPort` (`/health` check) | `b_network`, `c_security_groups`, `d_EC2_web_app` |

There are no `b_certificates`, `b_SES` or `f_DNS_Records` stacks yet,
because there's no domain, HTTPS or email so far. Reach the app at the ALB's
`LoadBalancerDnsName` output.

---

## Prerequisites

1. **AWS CLI** configured for `us-east-1`.
2. **EC2 key pair** `My_Event_365_SSH_keys` in the region (override with
   `KeyPairName`). SSH can't reach the private instance, so use SSM Session
   Manager, but the key pair still has to exist for the launch to work.
3. **GitHub Personal Access Token** that can read
   `mattwalker75/mmoa_web_app`, passed to the EC2 stack as `GitHubToken`.
   It's the only NoEcho parameter and has no default.

---

## Deploy

```bash
cd templates

aws cloudformation deploy --stack-name mmoa-network \
  --template-file b_network.yaml

aws cloudformation deploy --stack-name mmoa-iam \
  --template-file b_IAM.yaml --capabilities CAPABILITY_NAMED_IAM

aws cloudformation deploy --stack-name mmoa-security-groups \
  --template-file c_security_groups.yaml

aws cloudformation deploy --stack-name mmoa-ec2-web-app \
  --template-file d_EC2_web_app.yaml \
  --parameter-overrides GitHubToken=ghp_xxx...

aws cloudformation deploy --stack-name mmoa-alb \
  --template-file e_ALB_mmoa.yaml
```

Tear down in reverse order (`e` → `d` → `c` → `b`).

---

## Database

The database is SQLite, set up the same way as `car_show_control_plane`: a
`.db` file at the repo root, built from `setup_sqlite_db.sql` by
`SETUP_SQLLITE_DB.sh`, with `database.engine = "sqlite"` in `config.json`.
The setup script **wipes** any existing DB before rebuilding it. User data
runs it once, on first boot. Never re-run it by hand on a live instance.

The DB is on the root EBS volume, which is deleted when the instance is
terminated. **The admin console's regular backups are the only durable copy.**

---

## Moving to HTTPS later

1. Add `b_certificates` (ACM cert for the mmoa domain) and `f_DNS_Records`
   (Route53 alias → `mmoa-alb-dns-name`), modeled on
   `car_show_management_infra`.
2. In `e_ALB_mmoa`, add a 443 listener with that cert forwarding to the
   target group, and change the :80 listener to a 301 redirect to HTTPS.
   The ALB SG already allows 443.
3. Update the app's `publicBaseUrl` to `https://…` (user data config edits).

---

## Resource names (defaults)

- VPC / network: `mmoa-vpc`, `mmoa-igw`,
  `mmoa-subnet-public1-us-east-1a`, `mmoa-subnet-public2-us-east-1b`,
  `mmoa-subnet-private1-us-east-1a`, `mmoa-subnet-private2-us-east-1b`,
  `mmoa-nat-eip-public1-us-east-1a`, `mmoa-nat-public1-us-east-1a`,
  `mmoa-rtb-public`, `mmoa-rtb-private1-us-east-1a`,
  `mmoa-rtb-private2-us-east-1b`
- IAM: `mmoa_ec2_web_app` (role + profile), `mmoa_ec2_adminconsole`
  (role + profile), `mmoa_admin_console` (managed policy)
- Security groups: `mmoa_ec2_web_app_sg`, `mmoa_alb_inbound_sg`
- EC2: `mmoa_web_app`
- ALB: `mmoa-webapp-alb`, `mmoa-webapp-tg`
