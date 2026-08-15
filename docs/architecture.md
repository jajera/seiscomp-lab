---
title: Architecture
layout: default
nav_order: 2
---

# Architecture
{: .no_toc }

One VPC, one public subnet, one Ubuntu host. No NAT Gateway. Outbound internet uses a public IPv4 address on the instance.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

- TOC
{:toc}

---

## Why this layout

A learning box needs outbound HTTPS (apt, gsm) and outbound TCP 18000 (GEOFON SeedLink). It also needs a way in that is not SSH open to the world.

NAT Gateway was skipped on purpose. The instance sits in the **public** subnet with a public IP (or Elastic IP) and is administered with **SSM Session Manager**.

## Network

| Resource | Suggested name | CIDR / notes |
|---|---|---|
| VPC | `seiscomp-lab` | `10.81.0.0/16`, DNS hostnames and DNS support on |
| Public subnet | `seiscomp-lab-public-2a` | `10.81.1.0/24` in `ap-southeast-2a`, map public IPv4 on launch |
| Private subnet | `seiscomp-lab-private-2a` | `10.81.11.0/24`, no NAT (reserved, unused) |
| Internet gateway | `seiscomp-lab-igw` | attached |
| Public route table | `seiscomp-lab-public` | `0.0.0.0/0` → IGW |
| Private route table | `seiscomp-lab-private` | local only |
| Security group | `seiscomp-lab-aio` | egress all; inbound TCP 3389 from **your** `/32` only |
| IAM instance profile | `seiscomp-lab-ssm` | `AmazonSSMManagedInstanceCore` |

Tags: `Project=seiscomp-lab`.

### Not opened

- Inbound SSH (`22`) — use SSM
- Inbound SeedLink (`18000`) — this host **pulls** data; it does not publish
- Inbound MariaDB (`3306`) or scmaster (`18180`) or FDSNWS (`8080`)
- SSM VPC interface endpoints — public subnet plus IGW is enough for SSM

## On the host

```text
GEOFON geofon.gfz.de:18000
        |
        v
   seedlink  (0.0.0.0:18000, SG does not allow inbound)
        |
        +--> slarchive  (SDS, 7 days)
        |
        v
   scautopick --> scautoloc --> scamp --> scmag --> scevent --> scalert
        |
        v
   scmaster  (127.0.0.1:18180)
        |
        +--> MariaDB seiscomp (127.0.0.1:3306)
        +--> fdsnws (127.0.0.1:8080)
        +--> scqc, scevtlog
        +--> GUIs over XFCE / xrdp (3389)
```

systemd unit `seiscomp.service` starts SeisComP after MariaDB.

## Cost
{: .cost }
> Rough on-demand Sydney: `t3.xlarge` about $0.20/hour plus 150 GB gp3. An Elastic IP is free while associated with a **running** instance and bills if the instance is stopped and the EIP stays allocated.
