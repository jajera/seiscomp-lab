# seiscomp-lab

Single-host **SeisComP** learning lab on AWS Ubuntu. One EC2 runs acquisition, messaging, the catalog database, automatic processing, FDSNWS, and an XFCE desktop. Packages come from the **public** gsm repository (AGPL). No gempa commercial modules, no containers.

|                 |                                                         |
| --------------- | ------------------------------------------------------- |
| AWS CLI profile | `sandbox`                                               |
| Region          | `ap-southeast-2`                                        |
| OS              | Ubuntu 24.04 LTS                                        |
| SeisComP        | 7.3.1 (public gsm) + `world-minimal` maps               |
| Access          | SSM Session Manager; RDP to XFCE (security group `/32`) |

## Docs

📖 **[Published documentation site](https://jajera.github.io/seiscomp-lab/)** — architecture, SeisComP layout, desktop, walkthrough, prove, troubleshooting.

| Doc                                                    | Purpose                                    |
| ------------------------------------------------------ | ------------------------------------------ |
| [docs/index.md](docs/index.md)                         | Site overview and reading order            |
| [docs/architecture.md](docs/architecture.md)           | VPC, security group, processes on the host |
| [docs/seiscomp.md](docs/seiscomp.md)                   | gsm, MariaDB, inventory, bindings, modules |
| [docs/desktop.md](docs/desktop.md)                     | XFCE, xrdp, Windows RDP, launchers, toasts |
| [docs/walkthrough.md](docs/walkthrough.md)             | Manual deploy, prove, destroy              |
| [docs/prove.md](docs/prove.md)                         | Checks for SeedLink, FDSNWS, GUIs          |
| [docs/troubleshooting.md](docs/troubleshooting.md)     | SSM shell, xrdp, gsm prompts               |
| [docs/reference.md](docs/reference.md)                 | File map and example inventory             |
| [docs/walkthrough-brief.md](docs/walkthrough-brief.md) | Authoring notes (not published)            |

The site is built with [just-the-docs](https://just-the-docs.com/) and deploys from `.github/workflows/docs.yml`.

### Preview docs locally

```bash
./scripts/docs-serve.sh
```

Open [http://127.0.0.1:4000/seiscomp-lab/](http://127.0.0.1:4000/seiscomp-lab/).

## Layout

```text
docs/                     # just-the-docs site (GitHub Pages)
  _config.yml
  Gemfile
  index.md
  architecture.md
  seiscomp.md
  desktop.md
  walkthrough.md
  prove.md
  troubleshooting.md
  reference.md
  walkthrough-brief.md    # authoring notes, excluded from the build
scripts/docs-serve.sh     # local Jekyll preview only
```

There is no installer and no Terraform in this repo. You type AWS CLI on your laptop and commands on the instance over SSM.

## Prerequisites

- AWS CLI v2 and profile `sandbox`
- [Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)
- Permission to create a VPC, EC2, IAM instance profile, and (optional) Elastic IP in `ap-southeast-2`

The `sandbox` profile in this lab has **no** region in `~/.aws/config`. Always set `AWS_DEFAULT_REGION=ap-southeast-2`.

## Deploy

Follow [docs/walkthrough.md](docs/walkthrough.md): VPC → Ubuntu 24.04 → gsm → MariaDB → inventory/bindings → desktop → prove.

```bash
export AWS_PROFILE=sandbox AWS_DEFAULT_REGION=ap-southeast-2 AWS_PAGER=""
```

## Prove it

On the instance as `sysop`:

```bash
seiscomp check
slinktool -Q localhost
curl -sS 'http://127.0.0.1:8080/fdsnws/station/1/query?net=GE&level=station&format=text'
```

From Windows: Remote Desktop to the Elastic IP, session **Xorg**, user `sysop`. Password is SSM parameter `/seiscomp-lab/sysop-rdp-password` (not in git). See [docs/desktop.md](docs/desktop.md).

## Destroy

Stop or terminate the instance, release the Elastic IP, then delete the VPC stack (subnets, IGW, security group, IAM instance profile). Details in [docs/walkthrough.md](docs/walkthrough.md#destroy).

## Cost note

The walkthrough uses `t3.xlarge` and 150 GB gp3. Stop the instance when idle. An Elastic IP costs money if it stays allocated to a **stopped** instance.
