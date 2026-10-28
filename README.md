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
regularly by the admin console (plus a daily on-instance snapshot — see
[Database](#database)).

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
| App listen port (`config.json` → `port`) | `c_security_groups` → `AppPort` (exported as `mmoa-app-port`; imported by the ALB target group **and** by the EC2 user data, which seds it into `config.json`) | `3004` |
| App directory on the instance | `d_EC2_web_app` → `/home/<WebAppUserId>/mmoa_web_app` (the admin console's `ec2.appPath`) | `/home/mmoa/mmoa_web_app` |
| OS user that runs the app | `d_EC2_web_app` → `WebAppUserId` (the admin console's `ec2.runAsUser`) | `mmoa` |
| SQLite setup script | `d_EC2_web_app` → `DbSetupScript` | `SETUP_SQLLITE_DB.sh` |
| App start script | `d_EC2_web_app` → `StartScript` (first boot **and** the systemd unit; the unit's stop side is `STOP_APP.sh`) | `START_APP.sh` |
| Node.js | `d_EC2_web_app` → `NodeVersion` (nvm, per app user; the app needs ≥ 20.19 and its Angular 21 build supports 20 / 22 / 24) | `24` |
| Public URL baked into verification / reset links and Open Graph tags | `d_EC2_web_app` → `PublicBaseUrl` (optional; blank keeps `http://localhost:3004`) | blank |
| Health check path | `e_ALB` → `HealthCheckPath` (`GET /health`, 200, no DB access) | `/health` |
| `sqlite3` CLI on the host | `d_EC2_web_app` user data installs AL2023's `sqlite` package (3.40+). The admin console's `sqlite3 -json` queries need ≥ 3.33 | AL2023 |

User data seds the committed (scrubbed) `config.json` on first boot: `port`
(from the export above), `server.ssl` false, `server.trustProxy` 1,
`database.engine` sqlite, `aws.runningInAws` true, and `publicBaseUrl` when
`PublicBaseUrl` is set. The committed file already carries the first four
values, so those seds are belt-and-braces that keep the contract pinned on
the infra side. Change anything else afterwards from the admin console (Web
App Settings), then restart the app from its power bar.

Email is not set up yet, so the committed config has
`email.showLinksInBrowser: true`. With that on, verification and
password-reset links are shown on screen instead of only being emailed.
Turn it off once real email works. (When SES is switched on, add a
`ses:SendEmail` grant scoped to the identity to the `mmoa_ec2_web_app` role
in `b_IAM`, the way `car_show_management_infra` does for its control plane —
user data already sets `aws.runningInAws: true`, so no keys go on disk.)

---

## Architecture

```
                 Internet
                    │  HTTPS :443 missionmenofanna.com (HTTP :80 → 301 to HTTPS)
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
        │ SQLite (local, WAL)      │        │ config / logs / backups│
        │ + daily .backup timer    │        └────────────────────────┘
        └────────────┬─────────────┘
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
| [`a_DNS_base.yaml`](templates/a_DNS_base.yaml) | Route53 hosted zone for `missionmenofanna.com` with the records the domain has at GoDaddy today: Microsoft 365 (MX, verification + SPF TXT, `autodiscover` / `email` / `msoid` CNAMEs), Teams (`lyncdiscover` / `sip` CNAMEs, two SRVs), GoDaddy Payments (`pay` CNAME). The old GoDaddy site is not carried over (optional `LegacyWebsiteIps`). **Deployed**, unlike the car show's reference-only copy | none |
| [`b_certificates.yaml`](templates/b_certificates.yaml) | One ACM certificate, `missionmenofanna.com` + SAN `www.missionmenofanna.com`, DNS-validated in the zone; exports `mmoa-certificate-arn` | `a_DNS_base` **and** the nameserver switch |
| [`b_SES.yaml`](templates/b_SES.yaml) | SES domain identity (Easy DKIM, RSA 2048), MAIL FROM `mail.missionmenofanna.com`, and its records: 3 DKIM CNAMEs, MAIL FROM MX + SPF, `_dmarc` (`p=none`) | `a_DNS_base` **and** the nameserver switch |
| [`b_network.yaml`](templates/b_network.yaml) | Small VPC `10.1.0.0/24`, 2 public + 2 private `/26` subnets across `us-east-1a/b`, IGW, single NAT Gateway in AZ1, route tables | none |
| [`b_IAM.yaml`](templates/b_IAM.yaml) | `mmoa_ec2_web_app` role/profile (SSM core only). `mmoa_admin_console` **managed policy** (SSM Run Command to the instance tagged `mmoa_web_app` + read-only EC2/ELB). `mmoa_ec2_adminconsole` role/profile (SSM core + that policy). That role isn't attached to anything yet because the console runs off-AWS; attach the managed policy to the console's IAM user with `aws iam attach-user-policy` | none |
| [`c_security_groups.yaml`](templates/c_security_groups.yaml) | Web app SG `mmoa_ec2_web_app_sg` (self-ingress on `AppPort`) and ALB SG `mmoa_alb_inbound_sg` (80/443 from `0.0.0.0/0`) | `b_network` |
| [`d_EC2_web_app.yaml`](templates/d_EC2_web_app.yaml) | `mmoa_web_app`: single `t3.small` AL2023 instance, `8 GiB gp3` root, private subnet 1, SSM-managed. User data clones + configures + starts the app, installs `mmoa-web-app.service` (restart on reboot) and `mmoa-web-app-backup.timer` (daily SQLite snapshot) | `b_IAM`, `b_network`, `c_security_groups` |
| [`e_ALB.yaml`](templates/e_ALB.yaml) | `mmoa-webapp-alb` + HTTPS:443 listener (certificate from `b_certificates`) **forwarding** to `mmoa-webapp-tg`, which registers the instance on `AppPort` (`/health` check); HTTP:80 **redirects** to HTTPS | `b_network`, `c_security_groups`, `d_EC2_web_app`, `b_certificates` |
| [`f_DNS_Records.yaml`](templates/f_DNS_Records.yaml) | Apex + `www` A-alias records → the ALB | `a_DNS_base`, `e_ALB` |

`b_certificates`, `b_SES` and `f_DNS_Records` take the zone as a
`HostedZoneId` **parameter** (the `HostedZoneId` output of `a_DNS_base`), the
same way the car show stacks do. Until the domain is live, reach the app at
the ALB's `LoadBalancerDnsName` output.

---

## Prerequisites

1. **AWS CLI** configured for `us-east-1`.
2. **EC2 key pair** `My_Event_365_SSH_keys` in the region (override with
   `KeyPairName`). SSH can't reach the private instance, so use SSM Session
   Manager, but the key pair still has to exist for the launch to work.
3. **GitHub Personal Access Token** that can read
   `mattwalker75/mmoa_web_app`, passed to the EC2 stack as `GitHubToken`.
   It's the only NoEcho parameter and has no default. Make it a
   **fine-grained** token: GitHub → Settings → Developer settings →
   Personal access tokens → *Fine-grained tokens* → Generate new token;
   Repository access = *Only select repositories* → `mmoa_web_app`;
   Permissions → Repository permissions → **Contents: Read-only** (nothing
   else); pick a short expiry. A clone is the token's only job, so you can
   revoke it as soon as the stack is up (see the
   [security note](#github-token-handling) below).

---

## Parameters (`d_EC2_web_app`)

Everything except `GitHubToken` has a default; the other stacks only take
`ProjectName` plus their resource-name parameters.

| Parameter | Default | Notes |
|-----------|---------|-------|
| `ProjectName` | `mmoa` | Must match every other stack (exports / imports) |
| `WebAppInstanceName` | `mmoa_web_app` | Name tag; must match `WebAppInstanceName` on `b_IAM` (the admin console's SSM grant is scoped to it) |
| `InstanceType` | `t3.small` | t3.micro / small / medium / large |
| `AmiId` | latest AL2023 x86_64 (SSM public parameter) | Keep it AL2023 — user data relies on dnf, systemd and the 3.40+ `sqlite` package |
| `KeyPairName` | `My_Event_365_SSH_keys` | Must exist in the region |
| `RootVolumeSizeGiB` / `RootVolumeType` | `8` / `gp3` | Also holds the DB and its daily snapshots |
| `WebAppUserId` | `mmoa` | OS user that owns `/home/<user>/mmoa_web_app` and runs the app |
| `GitHubId` | `mattwalker75` | Username in the clone URL |
| `GitHubToken` | — (**required**, NoEcho) | Fine-grained PAT, read-only Contents on `mmoa_web_app` |
| `NodeVersion` | `24` | Installed with nvm as the app user; a major (`24`) or `major.minor`. Pinned instead of `--lts` so a newer LTS line can't land outside what the Angular build supports |
| `DbSetupScript` | `SETUP_SQLLITE_DB.sh` | Run once before the first start; skipped (with a warning) if missing |
| `StartScript` | `START_APP.sh` | Run by the systemd unit at every boot |
| `PublicBaseUrl` | blank | `http(s)://host[:port]`; blank keeps the committed `http://localhost:3004`. The ALB's DNS name isn't known until the `e_` stack exists, so either pass the real domain or set it later from the admin console and restart |
| `BackupRetentionDays` | `14` | Days of daily `.backup` snapshots kept in `/home/<user>/backups` |

---

## Deploy

Deploy in wave order; the EC2 stack imports from `b_IAM`, `b_network` and
`c_security_groups`, and the ALB stack imports the instance ID from the EC2
stack. Keep the PAT out of shell history by reading it into an env var first:

```bash
cd templates

aws cloudformation deploy --stack-name mmoa-network \
  --template-file b_network.yaml

aws cloudformation deploy --stack-name mmoa-iam \
  --template-file b_IAM.yaml --capabilities CAPABILITY_NAMED_IAM

aws cloudformation deploy --stack-name mmoa-security-groups \
  --template-file c_security_groups.yaml

read -rs GITHUB_PAT       # paste the fine-grained token; it will not echo
export GITHUB_PAT

aws cloudformation deploy --stack-name mmoa-ec2-web-app \
  --template-file d_EC2_web_app.yaml \
  --parameter-overrides GitHubToken="$GITHUB_PAT"
  # add PublicBaseUrl=https://www.example.org once there is a domain

aws cloudformation deploy --stack-name mmoa-alb \
  --template-file e_ALB.yaml

aws cloudformation describe-stacks --stack-name mmoa-alb \
  --query "Stacks[0].Outputs[?OutputKey=='LoadBalancerDnsName'].OutputValue" \
  --output text
```

The EC2 stack reports `CREATE_COMPLETE` as soon as the instance is running —
user data (clone, `npm install`, the Angular build, first start) keeps going
for several more minutes after that. The ALB target turns healthy once
`GET /health` answers on the app port. To watch first boot:

```bash
aws ssm start-session --target <instance-id>
sudo tail -f /var/log/cloud-init-output.log
```

Tear down in reverse order (`e` → `d` → `c` → `b`). The instance's root
volume — and the database on it — goes with the EC2 stack; take a backup from
the admin console first.

### What user data does (first boot only)

1. `yum update`; install `git`, `gcc-c++`, `make` (native npm modules) and
   `sqlite` (the `sqlite3` CLI; logs its version and warns if < 3.33).
2. `useradd -m <WebAppUserId>`; install nvm + Node `NodeVersion` as that user.
3. `git clone https://<GitHubId>:<token>@github.com/mattwalker75/mmoa_web_app.git`
   into `/home/<user>/mmoa_web_app` as that user, then rewrite the remote to
   the plain URL so the token is **not** kept in `.git/config`. A failed
   clone stops user data with an error (the usual cause is a wrong token).
4. `sed` `config.json`: `port` (the `mmoa-app-port` export), `server.ssl`
   false, `server.trustProxy` 1, `database.engine` sqlite,
   `aws.runningInAws` true, `publicBaseUrl` (if `PublicBaseUrl` is set);
   `chmod 600`; check it still parses.
5. Run `SETUP_SQLLITE_DB.sh` (builds `mmoa_web_app.db`).
6. Install `mmoa-web-app.service` — a oneshot unit that runs the repo's own
   `START_APP.sh` at boot and `STOP_APP.sh` at shutdown, as the app user,
   through a login shell so nvm's `node` is on `PATH`. It delegates to the
   scripts instead of supervising `node` itself so that the admin console's
   Start / Stop / Restart (the same scripts over SSM) and systemd never
   fight over the process.
7. Install `/usr/local/sbin/mmoa-web-app-backup.sh` plus
   `mmoa-web-app-backup.service` / `.timer` (daily, 09:00 UTC, catches up
   after a missed slot) and enable the timer.
8. `systemctl enable --now mmoa-web-app.service` (first start = `npm install`
   + SPA build + detached `node app.js`), then poll `GET /health` on the app
   port and print the result.

### GitHub token handling

The token rides in the clone URL once and is never echoed. It is removed
from the clone's `.git/config` immediately afterwards and `unset` in the
script. It is still visible for the life of the instance to anyone with
`ec2:DescribeInstanceAttribute` (the rendered user data) and under
`/var/lib/cloud/` on the box, exactly as with the car show control plane —
so use a short-lived read-only token and revoke it after the deploy, or move
to a Secrets Manager dynamic reference for production.

---

## Operating the instance

Everything below runs over SSM (`aws ssm start-session --target <id>`, or
the admin console, which wraps the same commands).

| What | Where |
|------|-------|
| First-boot log | `/var/log/cloud-init-output.log` |
| App log (requests, boot messages) | `/home/mmoa/mmoa_web_app/logs/app.log` (size-rotated by `START_APP.sh`) |
| START/STOP output from the unit | `journalctl -u mmoa-web-app` |
| Backup runs | `journalctl -u mmoa-web-app-backup` ; files in `/home/mmoa/backups/` |
| Start / stop / restart | admin console power bar, or `sudo -iu mmoa` → `cd mmoa_web_app` → `./RESTART_APP.sh` (also `sudo systemctl start mmoa-web-app` / `stop`) |

The admin console's power bar and `systemctl` are interchangeable: both end
up running `START_APP.sh` / `STOP_APP.sh`, which find the app by its PID
file and port. After a reboot the unit starts the app automatically.

### Updating the app

The clone's remote carries no credentials, so a pull needs a token again
(same fine-grained, read-only token as the deploy — paste it into a variable
so it stays out of shell history):

```bash
aws ssm start-session --target <instance-id>
sudo -iu mmoa
cd mmoa_web_app
read -rs GH_PAT; export GH_PAT
git pull "https://mattwalker75:$GH_PAT@github.com/mattwalker75/mmoa_web_app.git"
unset GH_PAT
./RESTART_APP.sh                   # config / backend-only changes
./RESTART_APP.sh --force-rebuild   # after dependency or frontend changes
```

The on-instance `config.json` is a locally modified tracked file, so if the
pull brings a change to `config.json` git refuses to overwrite it: `git stash`,
pull, `git stash pop`, settle any conflict in favour of the instance values,
and re-check them from the admin console's Web App Settings. User data does
not run again; the instance keeps whatever it has until you redeploy the EC2
stack (which replaces the instance — take a backup first).

---

## Database

The database is SQLite, set up the same way as `car_show_control_plane`: a
`.db` file at the repo root, built from `setup_sqlite_db.sql` by
`SETUP_SQLLITE_DB.sh`, with `database.engine = "sqlite"` in `config.json`.
The setup script **wipes** any existing DB before rebuilding it. User data
runs it once, on first boot. Never re-run it by hand on a live instance.

**WAL mode.** The app opens the file in WAL journal mode (so the admin
console's writes over SSM and the app's reads don't block each other). Live
data is therefore spread across `mmoa_web_app.db` **and** the sidecars
`mmoa_web_app.db-wal` / `-shm`: copying the `.db` file alone is **not** a
valid backup. Use SQLite's online backup instead, which is what the daily
timer does:

```bash
sqlite3 /home/mmoa/mmoa_web_app/mmoa_web_app.db ".backup /home/mmoa/backups/mmoa_web_app-$(date +%F).db"
```

**Daily snapshot.** `mmoa-web-app-backup.timer` runs that as the app user
every day at 09:00 UTC (03:00 / 04:00 Anna time), writes
`/home/mmoa/backups/mmoa_web_app-YYYY-MM-DD.db` (mode 600), and prunes files
older than `BackupRetentionDays` (14). Run one now with
`sudo systemctl start mmoa-web-app-backup.service`. Restoring one is a matter
of stopping the app, replacing `mmoa_web_app.db` (and deleting any stale
`-wal` / `-shm`), and starting it again.

Those snapshots are on the **same** root EBS volume as the live DB, and that
volume is deleted when the instance is terminated — they protect against bad
edits and botched updates, not against losing the instance. **The admin
console's regular backups (over SSM, into its own `backups/` folder off-box)
are the durable copy.**

---

## Domain, HTTPS and email (missionmenofanna.com)

The domain is registered at GoDaddy, which today also serves a Website
Builder site and Microsoft 365 email. The move keeps email working the whole
time. The old Website Builder site is **not** carried over (decided
2026-10-27): the domain goes straight to the web app.

**HTTPS note:** the old site sends
`Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`.
Browsers that visited it will load this domain **only over https** for up to
two years (the domain is not on the browsers' preload list, so this only
affects past visitors). For them the new site works once step 5 is done.

1. **Zone.** Deploy `a_DNS_base` with its defaults (email, Teams and `pay`
   records; no website records).
   ```bash
   aws cloudformation deploy --stack-name mmoa-dns-base --template-file a_DNS_base.yaml
   aws cloudformation describe-stacks --stack-name mmoa-dns-base \
     --query "Stacks[0].Outputs" --output table   # HostedZoneId + HostedZoneNameServers
   ```
2. **Site records.** Deploy `f_DNS_Records` (apex + `www` → the ALB). Safe
   before the nameserver switch — nothing in the zone is live yet.
   ```bash
   aws cloudformation deploy --stack-name mmoa-dns-records --template-file f_DNS_Records.yaml \
     --parameter-overrides HostedZoneId=<zone id>
   ```
3. **Nameservers.** At GoDaddy, replace the domain's nameservers with the four
   `HostedZoneNameServers`. Wait until `dig NS missionmenofanna.com` shows the
   `awsdns-*` servers (minutes to 48 hours). Email keeps working throughout.
   Transferring the *registration* away from GoDaddy is separate and can
   happen any time later.
4. **Certificate and SES** (both need step 3 live):
   ```bash
   aws cloudformation deploy --stack-name mmoa-certificates --template-file b_certificates.yaml \
     --parameter-overrides HostedZoneId=<zone id>
   aws cloudformation deploy --stack-name mmoa-ses --template-file b_SES.yaml \
     --parameter-overrides HostedZoneId=<zone id>
   ```
   The certificate stack waits until ACM validates (a few minutes once DNS
   has moved).
5. **HTTPS on the ALB.** Update the existing `e_ALB` stack with the
   current template (after `mmoa-certificates` is `CREATE_COMPLETE`). It adds
   the 443 listener with the certificate (`mmoa-certificate-arn`) forwarding
   to the target group and turns the :80 listener into a 301 redirect to
   HTTPS. The ALB SG already allows 443. Target group and instance are
   untouched.
6. **App settings** (Admin Console → Web App Settings), then restart:
   `publicBaseUrl` = `https://missionmenofanna.com` (links in emails and the
   Facebook / Open Graph share tags). Refresh an event link once in
   Facebook's Sharing Debugger.
7. **Email from the app.** Add a `ses:SendEmail` grant scoped to the
   `mmoa-ses-identity-arn` identity to the `mmoa_ec2_web_app` role in
   `b_IAM` (as `car_show_management_infra/b_IAM`'s `SesSend` does) and
   redeploy it. Make sure the account has SES production access (it is per
   account + region; in the sandbox SES only sends to verified addresses).
   Then set `email.transport` = `ses`, `email.from` = e.g.
   `noreply@missionmenofanna.com`, restart, send yourself a password reset —
   and once it arrives, turn **`email.showLinksInBrowser` off**.

To keep the old site answering during the move instead, deploy `a_DNS_base`
with `LegacyWebsiteIps=13.248.243.5,76.223.105.230`, skip step 2, and after
step 5 blank `LegacyWebsiteIps` and deploy `f_DNS_Records`.

Not carried over from GoDaddy: `_domainconnect` (GoDaddy's own helper).
There is no CAA record, which lets ACM issue the certificate. SES owns the
`_dmarc` record (there was none); Microsoft 365 keeps the apex MX and SPF.

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
- EC2: `mmoa_web_app`; on the instance: user `mmoa`, app in
  `/home/mmoa/mmoa_web_app`, snapshots in `/home/mmoa/backups`, systemd
  units `mmoa-web-app.service`, `mmoa-web-app-backup.service` /
  `.timer`, script `/usr/local/sbin/mmoa-web-app-backup.sh`
- ALB: `mmoa-webapp-alb`, `mmoa-webapp-tg`
- DNS / TLS / email: hosted zone `missionmenofanna.com`, certificate
  `mmoa-web-app-cert`, SES identity `missionmenofanna.com` with MAIL FROM
  `mail.missionmenofanna.com`
