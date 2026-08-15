---
title: Reference
layout: default
nav_order: 8
---

# Reference
{: .no_toc }

Repo layout and one example inventory from a completed lab. Recreate with the walkthrough; do not copy these IDs into new accounts.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

- TOC
{:toc}

---

## Repo files

| Path                        | Role                                           |
| --------------------------- | ---------------------------------------------- |
| `README.md`                 | Pitch, prereqs, short deploy / prove / destroy |
| `docs/_config.yml`          | just-the-docs                                  |
| `docs/index.md`             | Site overview                                  |
| `docs/architecture.md`      | VPC and on-host processes                      |
| `docs/seiscomp.md`          | gsm, DB, bindings, modules                     |
| `docs/desktop.md`           | XFCE and xrdp                                  |
| `docs/walkthrough.md`       | Commands                                       |
| `docs/prove.md`             | Checks                                         |
| `docs/troubleshooting.md`   | Pitfalls                                       |
| `docs/walkthrough-brief.md` | Authoring notes (excluded from Jekyll)         |
| `scripts/docs-serve.sh`     | Local preview                                  |

There is no Terraform and no SSM payload tree in this repository.

## SSM and IAM names

| Name                                         | Purpose                        |
| -------------------------------------------- | ------------------------------ |
| Instance profile `seiscomp-lab-ssm`          | `AmazonSSMManagedInstanceCore` |
| Parameter `/seiscomp-lab/sysop-rdp-password` | RDP password (SecureString)    |

## Example lab (2026-08-15)

Account `405087531484`, region `ap-southeast-2`, profile `sandbox`.

| Item           | Value                                          |
| -------------- | ---------------------------------------------- |
| VPC            | `vpc-0e960882bc774c4b3` `10.81.0.0/16`         |
| Public subnet  | `subnet-0ed7a31e9d6612c06` `10.81.1.0/24`      |
| Private subnet | `subnet-077acfd8de4c3a84c` `10.81.11.0/24`     |
| Security group | `sg-067cacbf78792d1ed`                         |
| Instance       | `i-0a05086a11cdbda17` Ubuntu 24.04 `t3.xlarge` |
| AMI (that day) | `ami-0eb87ec7ecb669408`                        |
| Elastic IP     | `15.135.124.32`                                |
| OS user        | `sysop`                                        |
| SeisComP       | 7.3.1 public + `world-minimal` 2025.328        |
| Stations       | GE.MORC, GE.RGN, GE.STU, GE.WLF                |

RDP was limited to one operator `/32`. Update the security group when that IP changes.
