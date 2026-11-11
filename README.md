# Wazuh SIEM on AWS: Single-Node Docker Install

Step-by-step commands to deploy **Wazuh 4.14.1** (manager, indexer, dashboard) in Docker on a private EC2 instance in AWS. Access is through AWS Systems Manager (SSM) Session Manager only: no SSH, no key pair, no public IP, and no open inbound ports.

This guide was written for a SOC team monitoring a serverless AWS application (API Gateway, Lambda, Cognito, RDS, S3). Serverless services cannot run agents, so Wazuh reads their logs from S3 and CloudWatch through its AWS module.

## Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Install steps](#install-steps)
- [Open the dashboard](#open-the-dashboard)
- [Daily commands](#daily-commands)
- [Troubleshooting](#troubleshooting)
- [After the install](#after-the-install)
- [Security notes](#security-notes)
- [Cost](#cost)

## Architecture

```
Analyst PC --(SSM tunnel, localhost:8443)--> AWS SSM --> Wazuh EC2 (private subnet)
                                                           |-- wazuh.manager
                                                           |-- wazuh.indexer
                                                           |-- wazuh.dashboard

Wazuh EC2 --> NAT gateway --> Internet (packages, Docker images, AWS APIs)
Wazuh EC2 --> S3 gateway endpoint --> S3 log bucket
```

Log sources the AWS module can read: CloudTrail, VPC Flow Logs, GuardDuty (from S3), and Lambda, API Gateway and RDS logs (from CloudWatch Logs).

## Prerequisites

| Item | Requirement |
| --- | --- |
| Instance | t3.xlarge (4 vCPU, 16 GB RAM) or larger |
| Disk | 50 GiB gp3, encrypted. The default 8 GiB is too small |
| OS | Ubuntu Server 22.04 or 24.04 LTS, x86_64 |
| Network | Private subnet with outbound TCP 443 and 80 through a NAT gateway |
| IAM role | Instance profile with `AmazonSSMManagedInstanceCore` |
| Metadata | IMDSv2 required. Hop limit **2** (Docker containers need it to use the instance role) |
| Security group | Outbound TCP 443 and 80 to `0.0.0.0/0`. No inbound rules needed for the dashboard |
| Network ACL | Outbound 443 and 80. Inbound 1024-65535 for replies (ACLs are stateless) |

Placeholders used below: `<INSTANCE_ID>` is the Wazuh EC2 instance ID, and `<PROFILE>` is your AWS CLI profile name.

Never type or paste real passwords into chat, tickets or screenshots.

## Install steps

Run every command as root, inside `tmux`, so a dropped session does not stop the install.

### 1. Connect to the server

EC2 > Instances > select the instance > Connect > Session Manager. Then open a proper root shell. Do not use `sudo su`, which can leave you in the SSM agent folder.

```bash
sudo -i
cd ~
```

### 2. Test the internet path

```bash
curl -sI https://packages.wazuh.com/key/GPG-KEY-WAZUH | head -1
```

Expected: `HTTP/2 200`. The site's home page returns 403, which is normal. A hang means the NAT, route table, network ACL or security group is blocking traffic.

### 3. Install tools and start tmux

```bash
apt-get update && apt-get install -y tmux git
tmux new -s wazuh
```

To reattach after a dropped session:

```bash
sudo -i
tmux attach -t wazuh
```

### 4. Install Docker Engine

```bash
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
```

Test it:

```bash
docker run --rm hello-world
docker compose version
```

### 5. Check the disk and set the kernel value

```bash
df -h /
sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
```

The disk must show about 48 to 50 GB. The indexer fails to start without the kernel value.

### 6. Download Wazuh and make certificates

```bash
cd ~
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.1
cd wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
ls config/wazuh_indexer_ssl_certs
```

Expected: `root-ca.pem`, `admin.pem`, `wazuh.indexer.pem`, `wazuh.manager.pem`, `wazuh.dashboard.pem` and their keys.

### 7. Set your own admin password (before the first start)

Use 8 to 64 characters with upper case, lower case, a digit and a symbol. Safe symbols: `.` `*` `-` `+` `?`. Avoid `$`, `#`, `|`, `&`, quotes, spaces and backslashes.

**7.1 Make the hash**

```bash
docker run --rm -ti wazuh/wazuh-indexer:4.14.1 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
```

Type your password when asked and copy the hash (it starts with `$2y$`).

**7.2 Put the hash in the indexer user file**

```bash
nano config/wazuh_indexer/internal_users.yml
```

In the `admin:` block, replace the `hash:` value with your hash inside double quotes. Keep `reserved: false`. Check it:

```bash
grep -n -A4 "^admin:" config/wazuh_indexer/internal_users.yml
```

**7.3 Make the Docker credentials match**

The manager and the dashboard both sign in to the indexer as `admin`, so change both `INDEXER_PASSWORD` lines. This reads the password without showing it and keeps it out of shell history:

```bash
read -rs PW
sed -i "s|INDEXER_PASSWORD=SecretPassword|INDEXER_PASSWORD=${PW}|" docker-compose.yml
unset PW
```

Check that neither line still says `SecretPassword`:

```bash
grep -n "INDEXER_PASSWORD" docker-compose.yml
```

`API_PASSWORD` and `DASHBOARD_PASSWORD` still hold defaults. Change them after the first login (see [After the install](#after-the-install)).

### 8. Start the stack

```bash
docker compose up -d
docker compose ps
df -h /
```

The first start downloads large images and takes about 5 minutes. All three services (`wazuh.manager`, `wazuh.indexer`, `wazuh.dashboard`) must show `Up`, and none may say `Restarting`.

Check the password works:

```bash
read -rs PW
curl -sk -u "admin:${PW}" https://localhost:9200
unset PW
```

Expected: JSON with a cluster name. `Unauthorized` means the hash and the compose password do not match.

## Open the dashboard

The server has no public IP, so use an SSM port-forwarding session (an SSM tunnel).

**Install on your PC:** AWS CLI v2 and the Session Manager plugin.

- https://awscli.amazonaws.com/AWSCLIV2.msi
- https://s3.amazonaws.com/session-manager-downloads/plugin/latest/windows/SessionManagerPluginSetup.exe

Reopen the terminal and check:

```powershell
aws --version
session-manager-plugin
```

**Sign in** with a separate profile so you do not overwrite other keys. SSO is better than long-lived keys.

```powershell
aws configure sso
aws sso login --profile <PROFILE>
```

Or with an access key:

```powershell
aws configure --profile <PROFILE>
```

Test it:

```powershell
aws sts get-caller-identity --profile <PROFILE>
```

**Start the tunnel:**

```powershell
aws ssm start-session --region eu-west-1 --profile <PROFILE> --target <INSTANCE_ID> --document-name AWS-StartPortForwardingSession --parameters "portNumber=443,localPortNumber=8443"
```

Wait for `Waiting for connections...` and leave the window open. Closing it ends the tunnel. Browse to **https://localhost:8443**, accept the self-signed certificate warning, and sign in as `admin`.

A new install shows 0 agents and 0 alerts. That is normal.

### Analyst access

Give each analyst their own accounts. Grant AWS permission for `ssm:StartSession` on the Wazuh instance and the `AWS-StartPortForwardingSession` document only. In Wazuh, create an internal user under **Indexer management > Security**, map it to read-only roles (`kibana_user` and `readall`), and map it to the `readonly` role under **Server management > Security > Roles mapping**. Test with a private browser window before handing it over. Keep `admin` for the SOC engineers.

## Daily commands

Run these in `/root/wazuh-docker/single-node`.

| Task | Command |
| --- | --- |
| Show status | `docker compose ps` |
| Stop the stack | `docker compose stop` |
| Start the stack | `docker compose up -d` |
| Restart the manager after a config change | `docker compose restart wazuh.manager` |
| Read a service log | `docker compose logs --tail=50 wazuh.indexer` |
| Open a shell in the manager | `docker compose exec wazuh.manager bash` |
| Manager log (inside the container) | `/var/ossec/logs/ossec.log` |
| Manager config (on the host) | `config/wazuh_cluster/wazuh_manager.conf` |
| Free disk space | `docker system prune -f` |

Containers restart on their own when the instance starts, because Docker is enabled.

## Troubleshooting

| Problem | Cause | Fix |
| --- | --- | --- |
| Session Manager shows "Not connected" | No IAM role, or no internet path | Attach the SSM role, reboot, check the NAT route and network ACL |
| `apt-get` hangs | Port 80 blocked | Allow outbound TCP 80 in the security group and the network ACL |
| "no space left on device" during pull | Root volume left at 8 GiB | Resize the EBS volume to 50 GiB, run `growpart /dev/nvme0n1 1` and `resize2fs /dev/nvme0n1p1` (`xfs_growfs /` for XFS), then `docker system prune -f` |
| Files seem missing | Shell started in the SSM agent folder | Run `sudo -i`, then `cd /root/wazuh-docker/single-node` |
| Indexer keeps restarting | `vm.max_map_count` not set | Run the step 5 commands, then `docker compose up -d` |
| Login fails | Hash and compose password do not match | Recheck the `admin:` hash and both `INDEXER_PASSWORD` lines |
| Dashboard says "not ready yet" | Indexer still starting | Wait 2 to 5 minutes and read the indexer log |
| `TargetNotConnected` (CLI) | Instance stopped or SSM agent offline | Start the instance and check Fleet Manager |
| `AccessDeniedException` (CLI) | Missing `ssm:StartSession`, or wrong profile or region | Check the profile, region and IAM policy |
| Port 8443 already in use | Another program uses it | Use `localPortNumber=8444` and browse to that port |

## After the install

1. Change the default `API_PASSWORD` (`wazuh-wui`) and `DASHBOARD_PASSWORD` (`kibanaserver`) using the Wazuh 4.14 documentation for Docker, and update `docker-compose.yml` to match.
2. Take an EBS snapshot of the server volume.
3. Create a central S3 log bucket, turn on CloudTrail, VPC Flow Logs, GuardDuty and RDS and API logging, and give the instance role read access to the bucket and the log groups.
4. Add the `aws-s3` module to `config/wazuh_cluster/wazuh_manager.conf`, then run `docker compose restart wazuh.manager`.
5. Narrow the security group: remove any inbound 443 rule, and limit ports 1514 and 1515 to the agent security group.
6. Write custom rules for your application's events, and test each one with the Wazuh `logtest` tool.

## Security notes

- Keep `docker-compose.yml` and `internal_users.yml` out of source control. They hold credentials. Use Docker secrets or AWS Secrets Manager for a longer-lived setup.
- Do not commit certificates, keys, account IDs, instance IDs or access keys.
- Never open port 22 or add a `0.0.0.0/0` inbound rule to the Wazuh security group.
- Use one account per analyst, with read-only roles.
- Pin the Wazuh image tag (4.14.1 here). Check [documentation.wazuh.com](https://documentation.wazuh.com) before moving to a newer release.

## Cost

The NAT gateway (about 1.1 USD per day) and the t3.xlarge (about 4.5 USD per day) are the main costs. Prices vary by region, so check the AWS Pricing Calculator.

- **Pause:** stop the instance, delete the NAT gateway, and release its Elastic IP. An unattached Elastic IP keeps billing.
- **Resume:** create a new NAT gateway, point the private route table at it, and start the instance.

## Reference

- Wazuh Docker deployment: https://github.com/wazuh/wazuh-docker
- Wazuh documentation: https://documentation.wazuh.com
- AWS Session Manager port forwarding: https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html
